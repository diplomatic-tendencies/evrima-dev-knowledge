# EVRIMA prime and elder mechanism

This is the working recipe for forcing a dino into "prime eligible" status from a UE4SS Lua mod on The Isle EVRIMA. It also documents the underlying mechanism the engine uses to decide whether a dino is prime, which matters because the naive "just set the bool" approach does not stick.

If you're building any kind of admin command, state-restore feature, or quest-reward path that needs to grant prime status, this is the document you want.

## The mechanism

`FEligiblePrimeElder` is a USTRUCT with 11 bool fields. Per `UHTHeaderDump-TheIsle/Public/EligiblePrimeElder.h`:

- 10 condition flags named `bPrimeCondition1` through `bPrimeCondition10`
- 1 cached output bool `bIsEligiblePrime`

The engine does not expose human-readable names for the 10 condition slots; the public surface is just the numbered conditions. From live observation (flipping each condition one at a time and watching effects) the rough mapping is partial:

| Slot | Observed objective | Stability |
|---|---|---|
| 1 | Visit sanctuary zone | per-life |
| 3 | Perfect diet (≥1% carb + protein + lipid simultaneously) | per-life |
| 7, 8, 10 | Breeding-path lifetime objectives | mostly persistent across respawn |
| 9 | Earned per-life (some per-life objective) | resets on respawn |
| 2, 4, 5, 6 | Never observed flipping | unknown / unimplemented |

Treat the slot-to-objective mapping as observational; only `bPrimeConditionN` is canonical.

The engine's rule is roughly "if at least 5 of the 10 `bPrimeConditionN` flags are true, set `bIsEligiblePrime` to true." The threshold check fires at specific gameplay moments: growth hitting 1.0, and some discrete events that re-evaluate prime eligibility.

Once eligible (5-of-10 threshold crossed), the cached `bIsEligiblePrime` bool gets locked in at the 75 percent growth mark. After that lock-in, the engine stops recomputing it from the condition flags. This is what makes prime "stick" for a dino that earned it: the condition flags can later go stale or be overwritten, but the locked bool remains true.

For a dino that has not crossed the threshold, `bIsEligiblePrime` is volatile. The engine recomputes it on every relevant tick. Setting the bool directly via `ServerSetPrimeEligible(true)` works for exactly one frame, then gets recomputed back to false from the 10 condition flags (most of which are false for a dino that hasn't actually earned prime).

## The naive approach that does not work

Early state-restore versions tried `pawn:ServerSetPrimeEligible(true)` to grant prime status. The bool flipped to true, then about a tick later flipped back to false. The restored dino would show prime briefly then lose it.

The interpretation that this was "the bool is read-only" or "the function is broken" was wrong. The actual behavior is "the engine recomputes the cached bool from the 10 condition flags, so setting the cached bool without also setting the condition flags is meaningless."

## The recipe that works

Force all 10 condition flags to true, force the cached eligible bool to true, and push the full struct via `SetEligiblePrimeElderData`. The engine sees all 10 flags as set (well above the 5-of-10 threshold), keeps `bIsEligiblePrime` true, and if the dino is at or past 75 percent growth, locks it in.

```lua
-- Force prime status by setting the full FEligiblePrimeElder struct.
-- The naive ServerSetPrimeEligible(true) does NOT stick because the engine
-- recomputes bIsEligiblePrime from the 10 condition flags. Forcing all 10
-- flags AND the cached bool together produces a state that the engine
-- accepts and locks in (at or past 75% growth).

local function forcePrime(pawn)
    if pawn == nil then return false, "no-pawn" end

    -- Get the live struct. POD fields, safe to access by name.
    -- (`pawn.EligiblePrimeElderData` direct field access also works.)
    local pe = pawn:GetEligiblePrimeElderData()

    -- Set all 10 condition flags. Field names match the UHT dump exactly:
    -- bPrimeCondition1..bPrimeCondition10.
    pe.bPrimeCondition1  = true
    pe.bPrimeCondition2  = true
    pe.bPrimeCondition3  = true
    pe.bPrimeCondition4  = true
    pe.bPrimeCondition5  = true
    pe.bPrimeCondition6  = true
    pe.bPrimeCondition7  = true
    pe.bPrimeCondition8  = true
    pe.bPrimeCondition9  = true
    pe.bPrimeCondition10 = true

    -- Set the cached output bool. Note: no trailing "Elder" — the dump's
    -- field is bIsEligiblePrime.
    pe.bIsEligiblePrime = true

    -- Push the struct. Single-arg setter; no bForceReplication. Engine
    -- handles replication via the standard property system.
    local ok = pcall(function() pawn:SetEligiblePrimeElderData(pe) end)
    return ok
end
```

All 10 conditions set true is permissive overkill — the threshold is 5 — but covers any engine evaluation path including ones that might check subsets we haven't enumerated.

## Capture for round-trip

For a state-restore mod, capture all 11 fields. Restore via the recipe above. The capture is just field reads on the live struct, all POD.

```lua
local pe = pawn:GetEligiblePrimeElderData()
state.prime = {
    isPrime = pe.bIsEligiblePrime,
    cond1   = pe.bPrimeCondition1,
    cond2   = pe.bPrimeCondition2,
    cond3   = pe.bPrimeCondition3,
    cond4   = pe.bPrimeCondition4,
    cond5   = pe.bPrimeCondition5,
    cond6   = pe.bPrimeCondition6,
    cond7   = pe.bPrimeCondition7,
    cond8   = pe.bPrimeCondition8,
    cond9   = pe.bPrimeCondition9,
    cond10  = pe.bPrimeCondition10,
}
```

For a restore mod, if the stored dino was prime, run the force-prime recipe. If it was not prime, restore the individual flag values; the engine will recompute the cached bool from them. Either approach round-trips cleanly.

## The lock-in mechanism

The 75 percent growth lock-in is what makes prime durable in normal gameplay. Once a dino crosses 75 percent growth with the 5-of-10 threshold met, the engine stops recomputing the cached eligible bool. After that point, you can clear the condition flags and the dino remains prime-eligible.

This matters for state-restore mods because the restored dino at any growth level may be above or below this 75 percent threshold. The force-prime recipe handles both cases: at less than 75 percent growth, the condition flags carry the prime state; at 75 percent or more, the cached bool gets locked in and the condition flags become irrelevant.

If you're testing prime persistence in your mod, do the test on a dino at exactly 75 percent growth and then again on the same dino at 0.74 and 0.76. The lock-in is the difference between those two cases.

## Setting prime status via chat command

The simplest admin command that grants prime to a target dino:

```lua
local function cmdSetPrime(senderSteam, args)
    if not isAdmin(senderSteam) then return end
    local targetSteam = args[1] or senderSteam
    local gm = findGameMode()
    if gm == nil then return notify(senderSteam, "no game mode") end
    local ctrl
    pcall(function() ctrl = gm:GetControllerBySteamId(targetSteam) end)
    if ctrl == nil then return notify(senderSteam, "target not online") end
    local pawn = livePawnFromCtrl(ctrl)
    if pawn == nil then return notify(senderSteam, "target has no pawn") end
    local ok = forcePrime(pawn)
    notify(senderSteam, ok and "prime set" or "prime set failed")
end
```

The `livePawnFromCtrl` helper is the rule-9a wrapper (filter for `pawn:GetAddress() ~= 0`). The `notify` helper is the rule-5 safe-notify pattern (call from a poll tick, not from inside the chat hook).

## Closing notes

The mistake to avoid is treating `bIsEligiblePrime` as the source of truth. It is a cached output, recomputed from the 10 `bPrimeCondition` flags. Setting it alone is undone the next time the engine recomputes.

If your mod's purpose is to grant prime as a reward (quest completion, admin grant, paid perk), use the force-prime recipe. It is the only path that survives the engine's recomputation logic across all growth levels.

If your mod's purpose is to capture and restore prime state without changing it, capture the full struct including all 10 condition flags plus the cached bool, restore the full struct via `SetEligiblePrimeElderData`. The engine handles the rest.
