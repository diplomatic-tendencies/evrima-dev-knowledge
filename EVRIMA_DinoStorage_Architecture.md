# DinoStorage architecture

DinoStorage is a state-restore mod for The Isle EVRIMA. It lets players store a snapshot of their current dino and later redeem the same dino back, preserving every restorable stat. The current production version (v021) handles vitals, growth, max-vitals, all 16 mutation slots, prime status, elder-replication-stacks (Life-tier counter), quest-mutation unlocks, all nutrients, and skin colors.

This document describes the architecture and the design decisions that produced the current version. The technical implementation of the underlying patterns (mutation persistence, elder stacks, customizer fields) lives in separate cookbook documents.

## Command surface

The mod exposes commands through the standard chat-hook interface:

- `!store` snapshots the player's current dino state to disk, then kills the live dino. Player respawns to spawn-zone select.
- `!redeem` mutates the player's freshly-spawned juvenile in-place using the most recent stored state for that steam.
- `!storeinfo` reports what's stored for the calling player (species, growth, when stored).
- `!storestatus` admin diagnostic showing all stored states server-wide.

The store side intentionally does not require any flags or arguments. The redeem side is similarly arg-free; if you have stored state, you can redeem it.

## Storage layout

```
Mods/DinoStorage/Saved/
├── storage.json              # Index of all stored states (one line per entry)
├── stored/
│   ├── <steam>.json          # Per-player stored state
│   └── <steam>.json
└── archive/                  # Optional: old states moved here on overwrite
    └── <steam>-<timestamp>.json
```

The per-player JSON file is the source of truth. The index file (`storage.json`) is a denormalized listing for fast enumeration. Write the per-player file first, then update the index. If a crash happens between those two writes, the index becomes inconsistent with the per-player files, but the per-player file is what `!redeem` actually reads. The index can be rebuilt at any time by scanning `stored/`.

## The transform-in-place architecture

The fundamental design choice is "transform-in-place, never respawn-with-customizer." The alternative path (use `RequestRespawn` to materialize the stored dino directly) is blocked because `RequestRespawn` crashes from Lua due to the `FCustomizerDataBase` by-value parameter (safety rule 4).

Transform-in-place works like this:

1. **Store phase**. Read every restorable stat into a flat JSON file. Then call `SetHealth(0)` to kill the live dino. The player goes to spawn-zone-select.

2. **Player action**. The player picks the same species at respawn. They land in the world as a freshly-spawned juvenile of that species. This step is intentional; it lets the engine handle the actual respawn flow (which has lots of internal state setup) instead of trying to replicate it.

3. **Redeem phase**. The mod detects the new pawn (via the presence registry plus a chat-hook command), then applies the stored state to the live pawn via scalar setters and POD-struct field writes. The juvenile becomes the restored adult within a few hundred milliseconds.

The benefit of this architecture is that every operation in the restore path uses safe primitives. No FCustomizerDataBase ever crosses a UFunction boundary by-value. The most "dangerous" operation in the path is `SetReplicatedMutationsData(struct, true)` which is well-tested and reliable.

## Stat application order

The application order is critical because of GAS-attribute auto-refill behavior (safety rule 8): every `SetGrowth` call wipes vitals to the new max. The order that survives all edge cases is documented in `EVRIMA_State_Restore_Cookbook.md`. In short:

1. Initial growth and vitals pass
2. Staged mutation slot apply (resetting growth, ramping up through unlock thresholds, setting each slot at its threshold)
3. Parent and Elder slot field-writes (no per-slot UFunction setters exist)
4. Nutrients
5. Re-apply vitals (because step 2 wiped them via growth changes)
6. Elder replication stacks
7. Skin colors

Steps 5, 6, and 7 happen in a deferred apply about 500 milliseconds after the bulk state writes. This is required for two reasons. First, `SetSlotNEquippedMutation` for quest mutations is rejected if called in the same tick as bulk state writes; the engine needs a settle window. Second, the field-write to `pawn.ReplicatedMutationsData` (which v021 uses as a replacement for `SetSlotNEquippedMutation` for quest mutations specifically) also tolerates being called shortly after the bulk writes, but reliability improves with the deferred apply.

## Mutation handling: the two-fix story

Mutation persistence ate roughly three days of debugging across two distinct bug rounds.

The first bug (fixed in v019) was that entombed Life 3 dinos came back with Life 1 effective values. Mutations were present in `ParentMutationSlot1-4` and `ElderMutationSlot*A/*B`, but the engine returned base values when game code queried `IsLifecycleMutationEquipped(name)`. Six hours of disassembly later, the root cause turned out to be `pawn.ElderReplicationStacks`, an int32 counter that nothing in the visible UFunction surface ever mentioned. The engine uses it as the actual tier gate; slot position (Parent vs Elder) is just bookkeeping.

The fix was six lines of code: capture `pawn:GetElderReplicationStacks()` into the JSON, restore via `pawn:SetElderReplicationStacks(n)` after the slot writes. Both UFunctions are clean native int32 in-out, safe from Lua. Verified live: Life 1 (stacks=0), Life 2 (stacks=1), and Life 3 (stacks=2) all round-trip with correct effective values.

The second bug (fixed in v021) was that quest-locked mutations (Reniculate Kidneys, Reinforced Tendons, Traumatic Thrombosis, Multichambered Lungs) did not equip on the restored pawn. The unlock list was empty after redeem, and even after restoring the unlock list, `SetSlotNEquippedMutation` silently rejected the equip calls.

The root cause was three layers stacked:

First, `pawn.MutationsRequirementsData.UnlockRequiredMutations` (TArray<FName>) was not captured by the original save logic. Session-fresh quest unlocks live only in this pawn-side array until some persistence event syncs them to player-data. `!store` kills the dino before that sync.

Second, `SetSlotNEquippedMutation` batches calls within a tick. Calling Slot1+Slot2+Slot3+Slot4 synchronously results in only Slot4 being equipped.

Third, quest mutations have an additional validation gate that silently fails on freshly-restored Life 2+ prime dinos. The gate appears to check `TIPlayerData.MutationRequirements` (a server-side player-data structure), which is not safely accessible from Lua.

The v021 fix replaces `SetSlotNEquippedMutation` entirely for the active-slot apply path. The mod writes directly to the slot FName field on the live `pawn.ReplicatedMutationsData` struct, then pushes via `SetReplicatedMutationsData(struct, true)`. Field-write bypasses the batching limit, the validation gate, and the post-bulk-state-write settle requirement. The unlock list is captured and restored separately via field-write plus `SetMutationRequirementsData`. The full implementation guide is in `EVRIMA_QuestMutation_Fix.md`.

After both fixes, every mutation case round-trips cleanly: regular slots, quest slots, Life 1, Life 2, Life 3, prime, parent inheritance, elder inheritance, and combinations.

## Skin handling

Skin colors are restored via the 7 FLinearColor fields on `pawn.CustomizerData` (`BodyColor`, `MarkingsColor`, `FlankColor`, `UnderbellyColor`, `Detail1Color`, `EyesColor`, `MaleDisplayColor`). The full pattern is in `EVRIMA_Customizer_Field_Map.md`. DinoStorage uses the per-field write path plus a `bumpPureBlack` helper to defeat the engine's pure-zero sentinel that would otherwise restore default colors after a relog.

## Lifecycle and chat hook integration

DinoStorage hooks the standard `/Script/TheIsle.TIPlayerController:GetChatMessage` for command parsing. Commands are deduplicated on `(sender, message)` with a 3-second window to handle the per-receiver hook fanout. Sender steam is extracted via the standard `GetSteamId():ToString()` chain on the sender controller (`ChatPlayerController` parameter, not `self`).

Heavy actions are deferred to a poll tick using the rule-5 hook-to-deferred pattern. The `!store` command queues a "kill this dino in 3 seconds" action; the `!redeem` command queues a "restore state to this pawn in 3 seconds" action. The deferred tick uses the presence registry's `livePawnFromCtrl` helper (rule 9a) to safely fetch the live pawn.

The presence registry pattern (`EVRIMA_Presence_Registry.md`) handles all online-player enumeration. DinoStorage itself does not iterate online players for the core store/redeem flow, but it does need it for admin diagnostic commands and for the per-mod heartbeat updates.

## JSON schema

The per-player stored state file has this top-level structure:

```json
{
  "version": "v021",
  "steam": "76561198XXXXXXX",
  "storedAt": 1716508800,
  "class": "/Game/.../BP_Tyrannosaurus_C",
  "growth": 1.0,
  "isFemale": false,
  "vitals": {
    "health": 9500, "stamina": 100, "hunger": 50, "thirst": 40,
    "oxygen": 100, "blood": 100, "lockedDamage": 0,
    "food": 600, "rottenValue": 0, "waterLevel": 100,
    "maxHunger": 100, "maxFood": 600, "maxThirst": 100, "maxStamina": 100
  },
  "location": {"x": 12345.0, "y": 67890.0, "z": 100.0,
               "pitch": 0, "yaw": 90, "roll": 0},
  "prime": {
    "eligible": true,
    "cond1": true,  "cond2": true,  "cond3": true,  "cond4": true,  "cond5": true,
    "cond6": true,  "cond7": true,  "cond8": true,  "cond9": true,  "cond10": true
  },
  "nutrients": {
    "carb": 100, "protein": 80, "lipid": 60, "bones": 40,
    "cannibal": 0, "magy": 0, "rottenFlesh": 0, "mushrooms": 0,
    "malnutrition": false
  },
  "mutations": {
    "Slot1": "Truculency", "Slot2": "Photosynthetic Tissue",
    "Slot3": "Reniculate Kidneys", "Slot4": "Traumatic Thrombosis",
    "ParentSlot1": "Epidermal Fibrosis", "ParentSlot2": "...",
    "ParentSlot3": "...", "ParentSlot4": "...",
    "ElderSlot1A": "...", "ElderSlot1B": "...",
    "ElderSlot2A": "...", "ElderSlot2B": "...",
    "ElderSlot3A": "...", "ElderSlot3B": "...",
    "ElderSlot4A": "...", "ElderSlot4B": "...",
    "elderStacks": 2
  },
  "unlockRequiredMutations": [
    "Reniculate Kidneys", "Reinforced Tendons",
    "Traumatic Thrombosis", "Multichambered Lungs"
  ],
  "skin": {
    "body":       {"R": 0.5, "G": 0.5, "B": 0.5, "A": 1.0},
    "markings":   {"R": 0.3, "G": 0.3, "B": 0.3, "A": 1.0},
    "flank":      {"R": 0.8, "G": 0.8, "B": 0.8, "A": 1.0},
    "underbelly": {"R": 0.1, "G": 0.1, "B": 0.1, "A": 1.0},
    "detail":     {"R": 0.5, "G": 0.5, "B": 0.5, "A": 1.0},
    "eyes":       {"R": 0.4, "G": 0.4, "B": 0.4, "A": 1.0},
    "breed":      {"R": 0.6, "G": 0.6, "B": 0.6, "A": 1.0},
    "skinVariation": 0.5,
    "patternIndex": 2
  }
}
```

The `version` field is critical. The schema changed twice between v016 (skin added), v019 (elder stacks added), and v021 (unlock list added). Code reads any version forward; missing fields take sensible defaults; the version field's main use is debugging "why did this old save break."

## Notable design choices

A few choices that look obvious in hindsight but were not at the time.

**Per-player files, not a single monolithic file.** Easier to debug ("what's stored for this player?"), easier to back up, easier to recover from partial corruption. The index file is denormalized for fast enumeration but is rebuildable.

**Version field from day one.** Even when there was nothing to version. The cost is zero; the value the first time you change the schema is enormous.

**Synchronous capture, deferred restore.** Capture runs in one tick (all reads, immediately followed by SetHealth(0)). Restore runs in a deferred tick (3-second delay after the `!redeem` command, then a further 500ms internal defer for the field-write phase). This is the pattern that survives the engine's various settle-window requirements.

**Same-species redemption only by default.** Cross-species redemption visually breaks HelloEvrima's nest tracking (the nest mesh disappears on species mismatch). The mod could allow it, but the user experience is bad. Same-species lock is the default.

**No respawn API ever called.** The whole architecture is built on the assumption that `RequestRespawn` is unreachable from Lua. This turned out to be correct and saved a lot of failed approach paths.

**The kill-and-respawn-naturally flow.** The store phase kills the live dino. The player respawns naturally via the engine's spawn UI. The redeem phase mutates the natural-respawn juvenile. This means the engine handles all the "spawn a new pawn" complexity; the mod only handles "mutate an existing live pawn." That choice removed roughly half the surface area of potential bugs.

## Closing notes

The mod has been in production across multiple servers for months. The complete bug count post-v021 is zero known issues. Pre-v021 (the entomb-bonus and quest-mutation bugs) the issue rate was about one bug per major lineage feature.

The hardest part of building it was not the obvious capture-and-restore loop. It was figuring out the engine's hidden state (elder stacks counter, quest-unlock validation gate, settle windows after bulk writes) that no public documentation describes. The patterns in `EVRIMA_State_Restore_Cookbook.md`, `EVRIMA_EntombBonus_Fix.md`, and `EVRIMA_QuestMutation_Fix.md` are the result. Anyone building a similar mod from scratch should read those three documents first; they would have saved most of the debugging cost.
