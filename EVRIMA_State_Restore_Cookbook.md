# EVRIMA dino state capture and restore cookbook

This is the working recipe for "snapshot a live dino, kill it, later put the player back as that exact same dino" on The Isle EVRIMA. Every pattern here was paid for with at least one server crash. Refer to `EVRIMA_Lua_Safety_Rules.md` for the underlying crash catalog.

The two companion docs `EVRIMA_EntombBonus_Fix.md` and `EVRIMA_QuestMutation_Fix.md` cover specific subtleties of the mutation system that bit me in two distinct rounds of debugging. Read those if your mod's purpose is mutation-related.

## Architecture: transform-in-place

Do not try to use `RequestRespawn`. It always crashes from Lua because the FCustomizerDataBase by-value parameter contains an FString, which UE4SS marshaling cannot copy without AVing (rule 4 of the safety doc).

The pattern that works:

1. `!store` (or whatever you call it) reads every restorable stat into JSON, then `SetHealth(0)` kills the live pawn. The player is dropped to the respawn screen.
2. The player picks the same species at respawn. They land in the world as a fresh juvenile of that species.
3. `!redeem` mutates the juvenile in-place via scalar setters and POD-struct field writes. No respawn API is called. The juvenile's pawn becomes the restored adult.

The reason this works where the respawn-API approach does not: every operation in the transform path uses safe primitives. No FCustomizerDataBase ever crosses a UFunction boundary by-value. The restore happens on a live pawn the player already controls, so there's no marshaling involved in delivery.

## What can be captured

Across roughly six months of iteration, these are the stat groups that survive a round trip:

| Stat group | Get pattern | Notes |
|---|---|---|
| Class path | `pawn:GetClass():GetFullName()` then strip the "BlueprintGeneratedClass " prefix | string |
| Growth | `pawn:GetGrowth()` | 0.0 to 1.0 |
| Vitals | `Get{Health,Stamina,Hunger,Thirst,Oxygen,Blood,LockedDamage,FoodValue,RottenValue,WaterLevel}()` | All floats. WaterLevel has no setter, capture-only |
| Max vitals | `Get{MaxHunger,MaxFoodValue,MaxThirst,MaxStamina}()` | Health, Oxygen, and Blood maxima derive from growth; no setters needed |
| Location, rotation | `pawn:K2_GetActorLocation()`, `pawn:K2_GetActorRotation()`, field reads | Safe scalar reads |
| Gender | `pawn:IsFemale()` | bool. No SetIsFemale; capture for warning only |
| Prime | `pawn:GetIsEligiblePrimeElder()` | bool. The volatile cached bool; see prime/elder mechanism doc |
| Nutrients | `pawn.NutrientsStruct.{CarbValue, ProteinValue, LipidValue, BonesValue, CannibalValue, MagyValue, RottenFleshValue, MushroomsValue, bMalnutrition}` | POD struct field access, safe |
| Active mutations | `pawn.ReplicatedMutationsData.{MutationSlot1..4, ParentMutationSlot1..4, ElderMutationSlot1A..4A, ElderMutationSlot1B..4B}` then `:ToString()` on each FName | All 16 slots restorable; see mutation section below |
| Elder replication stacks | `pawn:GetElderReplicationStacks()` | int32. Critical for mutation effective values; see below |
| Quest mutation unlocks | `pawn.MutationsRequirementsData.UnlockRequiredMutations` (TArray<FName>) | Required for quest mutations to re-equip on the restored pawn |
| Skin | 7 FLinearColor fields on `pawn:GetCustomizerData()` | See customizer field map doc |

FName extraction uses `:ToString()` on the FName userdata. The common mistake of `tostring(fname)` returns `"FNameUserdata: 0x..."` which is useless. Validate permissively; real mutation names like "Accelerated Prey Drive" contain spaces, only reject obvious garbage like quotes, backslashes, and control characters.

## Apply order matters

This is the part that took the longest to get right. The naive ordering "set everything in the order I captured it" produces a dino with empty mutation slots, wrong vitals, and broken prime status. The order below is the one that survived all the edge cases.

```lua
-- Step 1: initial vitals pass
pawn:SetGrowth(state.growth)
pawn:SetMaxHunger(state.maxHunger)  -- and the other maxes
pawn:SetHealth(state.health)        -- and the rest of the vitals
pawn:ServerSetPrimeEligible(state.isPrime)  -- volatile, see step 5

-- Step 2: active mutation slots via FIELD-WRITE (the v021+ way)
-- The legacy approach was a SetGrowth ramp + per-slot SetSlotNEquippedMutation
-- calls at each unlock threshold (~26/50/75/75 percent). That approach has
-- multiple silent-rejection edge cases (batching, quest-mutation validation
-- gates, post-bulk-state settle requirements) that bit DinoStorage twice.
-- See EVRIMA_QuestMutation_Fix.md for the full debugging story.
--
-- The v021+ pattern bypasses SetSlotN entirely. Write the FName directly into
-- the live ReplicatedMutationsData struct, push via SetReplicatedMutationsData,
-- and defer ~500ms after bulk state writes. This bypasses the batching limit,
-- the quest-mutation validation gate, and the settle requirement all at once.

LoopInGameThreadWithDelay(500, function()  -- ~500ms after the rest of the apply
    local liveMut = pawn.ReplicatedMutationsData
    if Slot1 then liveMut.MutationSlot1 = FName(Slot1) end
    if Slot2 then liveMut.MutationSlot2 = FName(Slot2) end
    if Slot3 then liveMut.MutationSlot3 = FName(Slot3) end
    if Slot4 then liveMut.MutationSlot4 = FName(Slot4) end
    pawn:SetReplicatedMutationsData(liveMut, true)  -- bForceReplication=true
end)

-- Step 3: parent and elder slots (no per-slot UFunction setters exist)
-- Field-write on the live struct using FName(s) userdata, then push via the
-- struct setter. Lua-string field-writes crash at 0x70; FName(s) field-writes
-- work.

local liveMut = pawn.ReplicatedMutationsData
local function setSlot(field, fnameStr)
    if not fnameStr or fnameStr == "" or fnameStr == "None" then return end
    local fname = FName(fnameStr)
    pcall(function() liveMut[field] = fname end)
end
setSlot("ParentMutationSlot1", state.mutations.ParentSlot1)
-- ...all 12 inherited slots (4 Parent + 4 ElderA + 4 ElderB)...
pawn:SetReplicatedMutationsData(liveMut, true)  -- bForceReplication=true

-- Step 4: nutrients
local live = pawn.NutrientsStruct
live.CarbValue = state.carbValue   -- etc
pawn:SetNutrientsStruct(live, true)

-- Step 5: re-apply GAS-attribute vitals AFTER mutation staging
-- Thirst, Hunger, Food, Stamina, Health are FGameplayAttributeData. Each
-- SetGrowth call earlier in the apply wiped them by recomputing max-stats and
-- refilling current to the new max. Re-set them now that no more SetGrowth
-- calls will run during this restore.
--
-- Note: the live mod actually re-applies vitals TWICE — once inside the
-- +500ms deferred mutation block (step 2 above) and once at the very end of
-- applyPawnState. The deferred apply is defensive against late-arriving
-- growth changes; the end-of-apply pass is the canonical one.

pawn:SetMaxHunger(state.maxHunger)  -- and the other maxes
pawn:SetHealth(state.health)         -- and the rest of the vitals

-- Step 6: elder replication stacks
-- See the dedicated section below. This is the lineage-tier counter and
-- without it, the mutation effective values stay at Life 1 even with
-- Parent and Elder slots populated correctly.

if state.elderStacks ~= nil and state.elderStacks > 0 then
    pcall(function() pawn:SetElderReplicationStacks(state.elderStacks) end)
end
```

## Mutation effective value computation

This took a full debugging cycle to figure out. Per-dino mutation effective values are not stored anywhere as a bumped number. The engine computes them dynamically when `IsLifecycleMutationEquipped(FName)` is queried. Inputs to the computation:

First, slot positioning: is the mutation in `MutationSlot1-4`, `ParentMutationSlot1-4`, or one of the eight `ElderMutationSlot*` fields?

Second, the value of `pawn.ElderReplicationStacks` (int32). This is the lineage-depth counter. **It is the actual tier gate.**

The static `LifecycleMutationsList` array on the pawn has three values per mutation: `EffectValue` (Life 1), `EffectValueLife2`, and `EffectValueLife3`. The engine returns one based on stacks:

- `stacks=0` returns `EffectValue` (Life 1) regardless of slot position.
- `stacks=1` returns `EffectValueLife2` for any slotted mutation.
- `stacks=2` returns `EffectValueLife3` for any slotted mutation.

Confirmed by toggling stacks via `SetElderReplicationStacks` on a dino with Truculency in `ParentMutationSlot1`:

- stacks=0: query returns 0.05 (base)
- stacks=1: returns 0.10 (Life 2)
- stacks=2: returns 0.15 (Life 3)

Same slot config, same parent setup, only stacks changed. Slot type (Parent vs Elder) does not gate the tier. It's just bookkeeping for which generation each mutation came from. The stacks counter does all the tier work.

`TIPlayerController.TemporaryEntumbStacks` exists but is not this counter. It's a transient transfer-time stash used during the entomb ceremony. Setting it does nothing for runtime queries; only `pawn.ElderReplicationStacks` matters.

If your save-restore code doesn't capture `ElderReplicationStacks` (a common oversight), an entombed Life 3 dino with all Parent and Elder slots populated comes back with Life 1 effective values. Mutations are technically present, just useless. This was the symptom that drove the v019 fix. The full standalone implementation is in `EVRIMA_EntombBonus_Fix.md`.

## Quest mutation persistence

Layered on top of the elder-stacks issue is the quest-mutation issue. Quest-locked mutations (Reniculate Kidneys, Reinforced Tendons, Traumatic Thrombosis, etc.) have a multi-layer persistence problem.

First layer: `UnlockRequiredMutations` (TArray<FName>) on `pawn.MutationsRequirementsData` is not captured by a naive save. Session-fresh quest unlocks live only in this pawn-side array until some persistence event syncs them to `TIPlayerData.MutationRequirements`. `!store` kills the dino before that sync, so the new spawn after `!redeem` hydrates from stale player-data and the fresh unlocks vanish.

Second layer: `SetSlotNEquippedMutation` silently rejects calls in several conditions. Within the same tick as bulk state writes (growth, vitals, etc.), the call gets rejected. The engine batches SetSlot calls within a tick; only the last in any rapid-fire batch actually commits. For quest mutations specifically, additional validation beyond `UnlockRequiredMutations` fails on freshly-restored Life 2+ prime dinos; the validation appears to check `TIPlayerData.MutationRequirements` which is not safely accessible from Lua.

Third layer: even after capturing the unlock list and restoring it before slot apply, the SetSlot batching and timing issues still apply.

The clean fix bypasses `SetSlotNEquippedMutation` entirely. Write directly to the slot FName field on the live `ReplicatedMutationsData` struct, then push via `SetReplicatedMutationsData(struct, true)`. Field-write does not run the engine's quest validation, does not batch, and tolerates being called shortly after bulk state writes. A 500-millisecond deferred apply via `LoopInGameThreadWithDelay` provides enough settle time after the preceding bulk state writes.

The full standalone implementation is in `EVRIMA_QuestMutation_Fix.md`. Reading that doc plus the elder-stacks one in `EVRIMA_EntombBonus_Fix.md` gives you everything needed to handle the mutation system end-to-end.

## Gotchas worth flagging

Gender cannot be restored. No `SetIsFemale` UFunction exists. Capture for warning purposes only. When the player picks the wrong gender at respawn, log the mismatch.

WaterLevel is capture-only. `GetWaterLevel()` exists; no `SetWaterLevel`. It gets restored to whatever the species default is on respawn.

Cross-species redemption (stored Triceratops, redeem as Allosaurus) breaks the nest visual because the engine cleans up species-mismatched nest meshes. The gameplay den still works (DenAura fires correctly), but the mesh disappears. The clean answer is to only allow same-species redemption. The hack answer is to warn the user and let them deal with the visual.

If a coexisting nest-persistence mod stores the owner class in its own JSON and reads it at `K2_PostLogin`, a cross-species mismatch between the stored class and the live char class can crash that mod (we've seen `0xffffffffffffffff` AVs in this case). Keep stored class and redemption class in sync to avoid the edge case.

Storage JSON schema rule of thumb: include a `version` field from day one. You will change the schema at least twice. Write all fields with sensible defaults so older saves still load. Use a flat top-level structure plus nested `nutrients` and `mutations` objects. Two-level nesting is the right balance for grep-ability and forward compatibility.

## Verified stats list

After all of this, the round-trippable stat list is:

- class, growth, all vitals plus maxes
- food (separate from hunger), rotten value
- all nine nutrient values (carbohydrate, protein, lipid, bones, cannibal, magy, rotten flesh, mushrooms, plus the bMalnutrition bool)
- all 16 mutation slots (Slot1-4 via per-slot UFunction setters with growth-staging, Parent and Elder via field-write with FName(s) plus SetReplicatedMutationsData)
- prime status, via the full FEligiblePrimeElder struct (see prime/elder doc, the volatile cached bool alone does not stick)
- skin: all 7 FLinearColor customizer fields, captured and restored per-field (see customizer field map doc)
- elder replication stacks for correct Life-tier effective values
- quest mutation unlocks list

Gender and water level are capture-only with no setter path; both are restored to the species default by the natural respawn pathway.

## Storage file location convention

The convention that worked is:

- Per-player stored state: `Mods/<YourMod>/Saved/stored/<steamid>.json`
- Index file: `Mods/<YourMod>/Saved/storage.json`

The index lets you avoid stat-ing every per-player file when listing what's stored. Keep the index in sync atomically (write to a temp file, rename) to survive server crashes mid-write.

## Closing notes

The single biggest mistake I made early was treating mutation effective values as state to capture rather than as something the engine computes dynamically from inputs. Once that flipped, the entomb-bonus fix took 20 lines of code. Before that, I was trying to find the "real" effective-value storage and getting nowhere.

The second-biggest mistake was assuming `SetSlotNEquippedMutation` would just work because it's the API the engine itself uses. It does work, but only in narrow conditions (within the right growth window, not in a batch, not for quest mutations on a freshly-restored Life 2+ prime dino). Replacing it with field-write plus struct-setter is the path that survives all the edge cases.

The third mistake was trusting `gm.AllPlayerControllers`. See the safety doc rule 3 and the presence registry doc for the fix.
