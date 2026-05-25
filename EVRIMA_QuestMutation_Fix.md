# Fix: quest-locked mutations lost across mutation-state save/restore

Implementation guide for fixing the "quest-unlocked mutations (Reniculate Kidneys, Reinforced Tendons, Traumatic Thrombosis etc.) disappear from active slots after save+restore" bug in any UE4SS Lua mod that snapshots and restores a player's mutation state on The Isle EVRIMA.

This is a companion to the entomb-bonus fix (`EVRIMA_EntombBonus_Fix.md`). They address different layers of the same broad problem: getting mutation state to survive a !store/!redeem cycle. The entomb-bonus fix handles the lineage-tier counter. This one handles the quest-unlock side of the same problem.

Both fixes are required to fully solve mutation persistence on quest-unlocked Life 2+ dinos.

---

## Symptom

1. Player completes a quest in normal gameplay (drink salt water repeatedly, eat certain items, whatever the trigger is). The engine adds the mutation FName to the player's unlock list.
2. Player equips that mutation in one of the four active slots.
3. Player runs your `!store` (or equivalent) on the dino.
4. Player respawns, then runs your `!redeem`.
5. The redeemed dino's `MutationSlot1-4` are empty (or partially empty) for the quest mutations specifically. Non-quest mutations equip fine. The mutation's effect (e.g. salt water drinking) does not work even though it was equipped pre-store.
6. The unlock list may also be empty or missing the recently-acquired entries.

If you instrumented this and watched the JSON captured by your save logic, the slot data probably IS in there. The bug is in the restore.

## What's actually broken

The mutation system has three independent layers, and all three need to round-trip cleanly:

1. **Active and inherited slot FNames** on `pawn.ReplicatedMutationsData` (16 fields: `MutationSlot1-4`, `ParentMutationSlot1-4`, `ElderMutationSlot1A-4A`, `ElderMutationSlot1B-4B`).

2. **`ElderReplicationStacks` (int32) on the pawn**. The lineage tier counter. Required for the engine to return the correct `EffectValueLifeN` for slotted mutations. See the companion entomb-bonus fix doc.

3. **`UnlockRequiredMutations` (`TArray<FName>`) on `pawn.MutationsRequirementsData`**. The list of quest-unlocked mutation names. Required for the engine to permit `SetSlotNEquippedMutation` calls for quest mutations.

A naive save/restore captures layer 1 and possibly layer 2 but misses layer 3. Without layer 3, the restore code's `SetSlotNEquippedMutation(FName("Reniculate Kidneys"))` calls silently return success but the engine ignores the equip because it sees that the mutation is not in `UnlockRequiredMutations`.

But that's not the only problem. Even after fixing layer 3, `SetSlotNEquippedMutation` has additional behaviors that cause it to silently reject for quest mutations on restored Life 2+ prime dinos:

- The engine batches `SetSlot` calls within a single tick. Only the LAST call in any rapid-fire batch actually commits. So calling `SetSlot1` + `SetSlot2` + `SetSlot3` + `SetSlot4` synchronously results in only `Slot4` being equipped.
- The engine needs a settle window after bulk state writes (growth, vitals, etc.) before accepting `SetSlot` calls. Immediate sync calls get rejected.
- Quest mutations have an additional validation gate that silently fails on freshly-restored Life 2+ prime dinos. I did not fully identify what this gate checks; possibly the save-side `TIPlayerData.MutationRequirements` rather than the pawn-side `MutationsRequirementsData`. It is not safely accessible from Lua.

The clean workaround for all three SetSlot problems: bypass `SetSlotNEquippedMutation` entirely and write directly to the slot FName field on the live `ReplicatedMutationsData` struct, then push via `SetReplicatedMutationsData(struct, true)`. Field-write does not run the engine's quest validation, does not batch, and tolerates being called shortly after bulk state writes.

## The fix

### Capture phase additions

In your !store handler, after the existing mutation slot capture, add the unlock list:

```lua
state.unlockRequiredMutations = {}
local mrData
pcall(function() mrData = pawn.MutationsRequirementsData end)
if mrData ~= nil then
    local arr
    pcall(function() arr = mrData.UnlockRequiredMutations end)
    if arr ~= nil then
        local n
        pcall(function() n = #arr end)
        if type(n) == "number" then
            for i = 1, n do
                local raw
                pcall(function() raw = arr[i] end)
                local s
                if raw ~= nil then
                    pcall(function() s = raw:ToString() end)
                end
                if type(s) == "string" and s ~= "" and s ~= "None" then
                    state.unlockRequiredMutations[#state.unlockRequiredMutations+1] = s
                end
            end
        end
    end
end
```

### Serializer addition

In your JSON write function, add a flat string array:

```lua
if state.unlockRequiredMutations ~= nil and #state.unlockRequiredMutations > 0 then
    local parts = {}
    for _, name in ipairs(state.unlockRequiredMutations) do
        parts[#parts+1] = string.format('"%s"', jsonEscape(name))
    end
    lines[#lines+1] = string.format('  "unlockRequiredMutations": [ %s ],', table.concat(parts, ", "))
end
```

### Deserializer addition

```lua
local unlockBlock = string.match(body, '"unlockRequiredMutations"%s*:%s*%[(.-)%]')
if unlockBlock then
    state.unlockRequiredMutations = {}
    for name in unlockBlock:gmatch('"([^"]+)"') do
        state.unlockRequiredMutations[#state.unlockRequiredMutations+1] = name
    end
end
```

### Restore phase: write the unlock list FIRST

Before any mutation slot work, write the unlock list back to the pawn so the engine has the validation data in place:

```lua
if state.unlockRequiredMutations ~= nil and #state.unlockRequiredMutations > 0 then
    local okMR, liveMR = pcall(function() return pawn.MutationsRequirementsData end)
    if okMR and liveMR ~= nil then
        local arr
        pcall(function() arr = liveMR.UnlockRequiredMutations end)
        if arr ~= nil then
            local currentN
            pcall(function() currentN = #arr end)
            currentN = type(currentN) == "number" and currentN or 0
            local existing = {}
            for i = 1, currentN do
                local raw
                pcall(function() raw = arr[i] end)
                if raw ~= nil then
                    local s
                    pcall(function() s = raw:ToString() end)
                    if type(s) == "string" then existing[s] = true end
                end
            end
            local added = 0
            for _, name in ipairs(state.unlockRequiredMutations) do
                if not existing[name] then
                    local okW = pcall(function() arr[currentN + 1 + added] = FName(name) end)
                    if okW then added = added + 1 end
                end
            end
            if added > 0 then
                pcall(function() pawn:SetMutationRequirementsData(liveMR) end)
            end
        end
    end
end
```

The merge logic (skip names already present) preserves any unlocks the engine hydrated from `TIPlayerData` on respawn. Pure overwrite would also work but might drop unlocks the dino legitimately had that the capture missed.

### Restore phase: replace SetSlotN with field-write

Remove your existing `SetSlot1-4EquippedMutation` calls. Replace with direct field-write on the live struct + a single `SetReplicatedMutationsData` push. Defer this by approximately 500ms so the engine has time to register your other state writes (growth, vitals) before processing the slot struct:

```lua
local steamSnap = steamId  -- capture for the closure
local slotsSnap = {
    { num = 1, str = state.mutations.Slot1 },
    { num = 2, str = state.mutations.Slot2 },
    { num = 3, str = state.mutations.Slot3 },
    { num = 4, str = state.mutations.Slot4 },
}

local handle
handle = LoopInGameThreadWithDelay(500, function()
    if handle and CancelDelayedAction then
        pcall(function() CancelDelayedAction(handle) end)
    end

    -- Re-derive the pawn via your presence registry pattern.
    -- Do NOT use a cached wrapper from the synchronous restore phase.
    local gm = findGameMode(); if gm == nil then return end
    local ctrl; pcall(function() ctrl = gm:GetControllerBySteamId(steamSnap) end)
    if ctrl == nil then return end
    local pawn2; pcall(function() pawn2 = ctrl:K2_GetPawn() end)
    if pawn2 == nil then return end
    local addr; pcall(function() addr = pawn2:GetAddress() end)
    if addr == nil or addr == 0 then return end

    local okMut, liveMut = pcall(function() return pawn2.ReplicatedMutationsData end)
    if not okMut or liveMut == nil then return end

    local written = 0
    for _, slot in ipairs(slotsSnap) do
        local s = slot.str
        if s ~= nil and s ~= "" and s ~= "None"
            and not s:find('["\\]') and not s:find("FNameUserdata") then
            local okF, fn = pcall(function() return FName(s) end)
            if okF and fn ~= nil and type(fn) ~= "string" then
                local fieldName = "MutationSlot" .. slot.num
                local okW = pcall(function() liveMut[fieldName] = fn end)
                if okW then written = written + 1 end
            end
        end
    end

    if written > 0 then
        pcall(function() pawn2:SetReplicatedMutationsData(liveMut, true) end)
    end
end)
```

Important details about this block:

- The 500ms defer is empirically the sweet spot. Sync (0ms) sometimes worked, sometimes did not, depending on how soon after a respawn the restore ran. 500ms reliably worked across every tested case. Going much higher (10s, 20s) also worked but the longer wait was noticeable to the player.
- `LoopInGameThreadWithDelay` is recurring per UE4SS semantics. Cancel via `CancelDelayedAction(handle)` inside the first fire so it does not repeat.
- Re-derive the pawn inside the deferred closure. Do not hold the wrapper from the synchronous restore phase; the pawn might have been destroyed in the meantime (rule 9a/9b territory). Use your presence registry pattern.
- `SetReplicatedMutationsData(struct, true)` with `bForceReplication=true` pushes the change to the client immediately. The client UI updates without a separate notify event.

### Ordering

The combined flow should be:

1. Synchronous restore of vitals, growth, max-vitals, prime data.
2. Synchronous restore of `UnlockRequiredMutations` (write to pawn struct + `SetMutationRequirementsData`).
3. Synchronous restore of Parent and Elder mutation slot FNames via existing field-write pattern + gated `SetReplicatedMutationsData`.
4. Synchronous restore of `ElderReplicationStacks` (per the entomb-bonus fix companion doc).
5. Deferred at +500ms: re-derive pawn, field-write `MutationSlot1-4`, push via `SetReplicatedMutationsData(struct, true)`. Also restore final growth and re-apply vitals here since any intermediate growth changes wipe them.

## Verification

Build a test workflow that:

1. Spawn a fresh dino, complete a quest to unlock a quest mutation.
2. Equip the quest mutation in MutationSlot1 via the in-game UI.
3. Read `pawn.ReplicatedMutationsData.MutationSlot1` from a diagnostic probe. Confirm it shows the quest mutation FName.
4. Read `pawn.MutationsRequirementsData.UnlockRequiredMutations` array. Confirm it includes the quest mutation FName.
5. Run `!store` and respawn fresh.
6. Run `!redeem`.
7. Wait at least 1 second.
8. Re-read both fields. Both should now contain the quest mutation FName as before.
9. In-game test the mutation's effect (drink salt water for Reniculate Kidneys, etc.). Confirm it works.

If step 8 shows the slot still empty, your field-write is not landing. Check that the `LoopInGameThreadWithDelay` callback fires and the `liveMut[fieldName] = fn` assignment runs. Check that `SetReplicatedMutationsData` runs after the field writes.

If step 8 shows the slot has the FName but step 9 fails, the engine has more validation than `UnlockRequiredMutations` alone. I did not find what other gate exists in this case but the most likely candidate is `TIPlayerData.MutationRequirements` which lives in player save data and is not safely accessible from Lua.

## Quirks worth knowing

- **Quest unlock FName casing matters**: `Reniculate Kidneys` (with the trailing 's') not `Reniculate Kidney`. `FName` is exact-match. Capture the actual string the engine uses by reading from `UnlockRequiredMutations` rather than hardcoding.
- **`TIPlayerController.TemporaryEntumbStacks` is a separate int32** on the controller. Do not confuse it with `pawn.ElderReplicationStacks`. The controller field is a transient transfer-time stash used by the engine during the natural entombment ceremony. It does nothing for runtime effective-value queries. Setting it via Lua has no effect.
- **`SetSlotNEquippedMutation` is the function the in-game UI uses** when a player equips a mutation via the mutation menu. It does work in that context. It only fails when called synchronously from a Lua-driven restore on a freshly-respawned pawn, especially for quest mutations and Life 2+ prime dinos. The field-write replacement is functionally equivalent for restore purposes but bypasses all the validation.
- **Field-write does not fire the engine's "you equipped a mutation" notification events**. If your mod or its consumers depend on those events to drive other behavior (UI updates, achievements, audit logs), you would need to fire them separately. For the standard restore use case nothing depends on them; the client UI updates via the `SetReplicatedMutationsData` replication push.
- **Pre-fix stored slots are not migratable**. Stored JSON files written before this patch do not contain `unlockRequiredMutations`. The deserializer returns nil. The restore code skips the unlock-list rewrite. Active slot field-writes will then fail for any quest mutation that wasn't already in the player's persistent unlock list. Affected players need to re-acquire the unlock via gameplay and re-store. There is no clean migration path because the unlock state was never captured.

## Reference UFunctions

All on `ATICharacterBase` unless noted:

| UFunction | Signature | Notes |
|---|---|---|
| `GetMutationRequirementsData` | `() const -> FMutationsRequirements` | Returns struct with `UnlockRequiredMutations: TArray<FName>` and `FractureMovementTime: float` |
| `SetMutationRequirementsData` | `(FMutationsRequirements Data)` | Commits the struct. No bForceReplication arg. |
| `SetReplicatedMutationsData` | `(FReplicatedMutationsData Data, bool bForceReplication)` | Pushes the slot struct. Use bForceReplication=true to update clients immediately. |
| `SetSlot1-4EquippedMutation` | `(FName Mutation)` | The natural-equip function. AVOID for restore use case (silently rejects in too many edge cases). |

Field-access (UE4SS Lua syntax):

| Field path | Notes |
|---|---|
| `pawn.ReplicatedMutationsData.MutationSlot1-4` | FName slot, writable via `field = FName("Name")` |
| `pawn.ReplicatedMutationsData.ParentMutationSlot1-4` | Same pattern |
| `pawn.ReplicatedMutationsData.ElderMutationSlot1A-4B` | Same pattern |
| `pawn.MutationsRequirementsData.UnlockRequiredMutations` | TArray<FName>. Append via `arr[n+1] = FName("Name")` then call `SetMutationRequirementsData` |

## Summary

Three small additions to your save/restore code: capture `unlockRequiredMutations` as a JSON FName array, restore it via field-write on the live struct + `SetMutationRequirementsData`, and replace your `SetSlotN` calls with direct field-write on `MutationSlotN` + a deferred `SetReplicatedMutationsData` push. Total addition is roughly 50 lines.

Combined with the entomb-bonus fix (`EVRIMA_EntombBonus_Fix.md` which adds `ElderReplicationStacks` capture/restore), this is the full mutation-state persistence solution. Verified across Life 1 single-slot, Life 1 multi-slot, and Life 2 prime multi-slot with quest+normal mutation mix. All effective values (including `PhotosyntheticTissueStatAdder` which reads as the Life 2 value `0.0700` on a restored Life 2 dino) work correctly post-restore.
