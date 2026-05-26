# Fix: entomb bonus lost across mutation-state save/restore

Implementation guide for fixing the "entombment bonus disappears after store and redeem" bug in any UE4SS Lua mod that snapshots and restores a player's mutation state on The Isle EVRIMA.

If your mod captures all 16 mutation slot FNames but a redeemed dino still shows Life 1 base values for its parent or elder mutations (e.g. Epidermal Fibrosis at 15 percent instead of 20 percent), this is your bug. The fix is 6 lines of code.

---

## Symptom

1. Player grows a dino, equips mutations.
2. Player entombs at 100 percent growth and respawns. The new dino correctly shows boosted mutation effects (Epidermal Fibrosis at 20 percent, Truculency at 10 percent, etc.).
3. Player runs your `!store` (or equivalent) on the entombed dino.
4. Player respawns as a fresh hatchling and runs your `!redeem`.
5. Redeemed dino has the same mutations in the correct `ParentMutationSlot1-4`, but the effective values silently drop back to Life 1 base (Epidermal Fibrosis at 15 percent, Truculency at 5 percent).

The captured JSON file has the correct slot names. The restored dino has the correct slot data. The engine still reports base values.

## Why this happens

The Isle's mutation system does not store the boosted value anywhere. The 20-percent Epidermal Fibrosis number is computed dynamically by the engine at query time. Two inputs feed that computation:

1. **The mutation slot data** on `pawn.ReplicatedMutationsData` (16 FName slots, MutationSlot1-4 + ParentMutationSlot1-4 + ElderMutationSlot1A-4A + ElderMutationSlot1B-4B). Most !store/!redeem mods already capture all 16 of these.

2. **`pawn.ElderReplicationStacks`** (int32 on `ATICharacterBase`). This is the lineage tier counter. Without it, the engine has no way to know which `EffectValueLifeN` template value to return.

The tier model is simple:

| `ElderReplicationStacks` | Engine returns | Player perception |
|---:|---|---|
| 0 | `EffectValue` | Life 1 (fresh hatchling, no entomb history) |
| 1 | `EffectValueLife2` | Life 2 (one entombment in the lineage) |
| 2 | `EffectValueLife3` | Life 3 (two entombments) |

The static `LifecycleMutationsList` array on the character class has all three values pre-baked. The engine picks which one to return based on the stacks counter alone. Slot positioning (parent vs elder) does NOT gate the tier. It is bookkeeping for which generation produced each mutation. The stacks counter is the source of truth.

When a player goes through a natural entombment cycle (UI prompt at 100 percent growth), the engine increments `ElderReplicationStacks` on the new spawn behind the scenes. When a custom !store kills the old dino and the player respawns naturally (fresh hatchling), the new spawn has `ElderReplicationStacks = 0`. Your !redeem writes the slot data but never sets the counter. Engine reads 0 and returns base values.

## The 6-line fix

### Capture side (in your !store handler, alongside the existing slot reads)

```lua
state.elderStacks = 0
pcall(function() state.elderStacks = pawn:GetElderReplicationStacks() end)
```

`GetElderReplicationStacks` is a clean native UFunction on `ATICharacterBase`. Returns an int32. No struct marshaling, no FName issues. The pcall is defensive against unexpected UE4SS edge cases but the call itself is known-safe.

### Serializer side (in your JSON write function)

```lua
if state.elderStacks ~= nil then
    -- format string matches whatever style your other int fields use
    lines[#lines + 1] = string.format('  "elderStacks": %d,', state.elderStacks)
end
```

### Deserializer side (in your JSON read function)

```lua
state.elderStacks = jsonReadNumber(body, "elderStacks")
-- or whatever helper you use to pull a number field by key
```

### Restore side (in your !redeem handler, AFTER you write the slot FNames and SetReplicatedMutationsData)

```lua
if state.elderStacks ~= nil and state.elderStacks > 0 then
    pcall(function() pawn:SetElderReplicationStacks(state.elderStacks) end)
end
```

`SetElderReplicationStacks` is also a clean native UFunction. Takes an int32 parameter. The check for `> 0` skips the call for Life 1 dinos since fresh hatchlings default to 0 and there is no need to overwrite that.

That is the entire fix. 4 lines in capture and restore, 2 lines in the JSON serializer.

### Ordering matters

Call `SetElderReplicationStacks` AFTER you write the slot FNames and AFTER your `SetReplicatedMutationsData(struct, true)` call. The engine pairs the counter with the slot data when computing effective values, so the slots need to be in place first.

### Capture happens before the dino dies

If your !store kills the dino as part of capture (via `SetHealth(0)` etc.), make sure you read `GetElderReplicationStacks` BEFORE the kill. Once the pawn is destroyed, the wrapper goes stale (rule 9a/9b territory) and the read will either return garbage or fire a delayed access violation.

## Verification

To confirm the fix is working, set up a diagnostic mod that exposes the engine's effective-value query through a chat command:

```lua
RegisterHook("/Script/TheIsle.TIPlayerController:GetChatMessage",
    function(self, newTextParam, senderControllerParam, ...)
        -- ...unwrap params, check for your debug command prefix...
        local pawn = senderCtrl:K2_GetPawn()
        if pawn == nil then return end

        local stacks = pawn:GetElderReplicationStacks()
        local effective = pawn:IsLifecycleMutationEquipped(FName("Truculency"))
        print(string.format("stacks=%d effective=%f\n", stacks, effective))
    end)
```

Then run a full round trip:

1. In game, equip 4 mutations on a fresh dino. Grow to 100 percent. Entomb via the in-game UI.
2. After respawn, your debug command should show `stacks=1` and `effective=0.10` for any of the 4 mutations you had equipped (now in Parent slots).
3. Run `!store` and respawn as a hatchling.
4. Run `!redeem`.
5. Your debug command should still show `stacks=1` and `effective=0.10`.

If step 5 matches step 2, the fix is working. If step 5 shows `stacks=0` and `effective=0.05`, the restore is not landing correctly.

The same test extends to Life 3. Manually set `pawn:SetElderReplicationStacks(2)` after an entomb, then verify Truculency returns 0.15 (the `EffectValueLife3` template value).

## Edge cases

### Pre-fix stored slots

If your mod has stored slots created before this patch, those JSON files do not contain the `elderStacks` field. The deserializer returns nil, your restore skips the `SetElderReplicationStacks` call, and the redeemed dino comes back as Life 1 regardless of what was stored. This is the same broken behavior as before the patch, not a new regression. New stores after the patch round-trip cleanly. There is no automatic migration. If a player needs their stored dino to come back as Life 2 or Life 3 and the slot was saved pre-patch, they will have to entomb a fresh dino and re-store it.

### Stacks higher than 2

The static template has Life 1 / Life 2 / Life 3 values. Setting `ElderReplicationStacks` to a value above 2 has not been tested yet .

### Multiple lineage chains via mating

The 16 slot fields include `ElderMutationSlot*A` and `ElderMutationSlot*B`. The A and B variants represent the two grandparent lineages (one per parent). They are pure bookkeeping for which mutations came from which grandparent and do not change the tier resolution. If your !store/!redeem captures and restores all 16 slot names, the A/B distinction is preserved automatically.

### The transient counter on the controller

`TIPlayerController` has a separate field called `TemporaryEntumbStacks` (also int32). Do NOT confuse this with `pawn.ElderReplicationStacks`. The controller's `TemporaryEntumbStacks` is a transient transfer-time stash used by the engine during the natural entombment ceremony to pass state from the dying dino to the new spawn. It has no effect on runtime effective-value queries. Setting it via Lua does nothing useful. The fix needs the pawn-side counter, not the controller-side one.

### `IsLifecycleMutationEquipped` is the canonical query

If your mod or its consumers (a bot, a website, an in-game UI display) needs to look up effective mutation values from outside the engine's normal flow, use `pawn:IsLifecycleMutationEquipped(FName(name))`. It returns a float that is the actual effective value the engine uses for gameplay. Returns 0 if the mutation is not in any slot. Returns the appropriate `EffectValueLifeN` if it is.

## Why DinoStorage and similar mods missed this

The engine's mutation system is split across two data sources, with the slot data on the pawn (`ReplicatedMutationsData`) and the tier counter as a sibling int property on the same pawn (`ElderReplicationStacks`). They are not in the same struct. A mod that focused on the visible slot data is easy to write without noticing the counter exists. The struct contains 16 FName slot fields plus 7 stat-adder floats and a bool, all related to mutation gameplay, so it looks complete.

The counter only matters when the engine queries effective values, and it queries them silently. There is no error log when the counter is zero. The dino just shows base values, and a player who has not tested the entomb cycle end-to-end will not notice anything is wrong until they hit Life 2 mutations that should be visibly stronger.

## Verified behavior table

The following was verified live on a dev server on 2026-05-23 against The Isle EVRIMA build with UE4SS 3.0.1 Beta:

| Slot configuration | `ElderReplicationStacks` | `IsLifecycleMutationEquipped(Truculency)` returns |
|---|---:|---:|
| Truculency in ParentMutationSlot1 | 0 | 0.05 (base) |
| Truculency in ParentMutationSlot1 | 1 | 0.10 (Life 2) |
| Truculency in ParentMutationSlot1 | 2 | 0.15 (Life 3) |
| No Truculency in any slot | any | 0 (not equipped) |
| Truculency in MutationSlot1 only | 0 | 0.05 (base) |

Setting stacks via `SetElderReplicationStacks` from Lua produced immediate visible effect on the next `IsLifecycleMutationEquipped` query, with no delay and no need to trigger any refresh function.

`SetReplicatedMutationsData(struct, true)` writes the slot data and replicates to clients but does NOT cause the engine to recompute effective values. The effective-value computation reads the slot fields and the stacks counter at query time, so any write to either is immediately visible without a cache invalidation step.

## Reference UFunctions

All on `ATICharacterBase`:

| UFunction | Signature | Notes |
|---|---|---|
| `GetElderReplicationStacks` | `() const -> int32` | Read the current tier counter |
| `SetElderReplicationStacks` | `(int32 NewStack) -> void` | Write the tier counter, immediate effect |
| `IsLifecycleMutationEquipped` | `(FName MutationToCheck) const -> float` | Returns effective value or 0 if not slotted anywhere |
| `IsElder` | `() const -> bool` | Returns true if stacks indicate an entombed lineage (exact threshold not verified) |
| `GetAllEnabledLifecycleMutations` | `() const -> TArray<FMutationsData>` | Returns the static template with all 3 EffectValueLifeN values per mutation |
| `SetReplicatedMutationsData` | `(FReplicatedMutationsData Data, bool bForceReplication)` | Writes the slot struct, replicates to clients, does NOT trigger cache rebuild (none needed) |

Note that `IsLifecycleMutationEquipped` returns a float despite the misleading name. Treat it as `GetMutationEffectiveValue`.

## Summary

The fix is 6 lines and works because the engine's effective-value query reads both the slot data and the stacks counter at every call. Any persistence layer that captures and restores both will produce a correct round trip. The slot capture was already in place in most existing implementations. The stacks counter capture and restore is the addition that closes the gap.
