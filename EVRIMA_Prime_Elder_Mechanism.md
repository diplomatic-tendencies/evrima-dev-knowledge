# EVRIMA prime and elder mechanism

This is the working recipe for forcing a dino into "prime eligible" status from a UE4SS Lua mod on The Isle EVRIMA. It also documents the underlying mechanism the engine uses to decide whether a dino is prime, which matters because the naive "just set the bool" approach does not stick.

If you're building any kind of admin command, state-restore feature, or quest-reward path that needs to grant prime status, this is the document you want.

## The mechanism

`FEligiblePrimeElder` is a USTRUCT with 11 fields. 10 of them are bool flags for individual prime conditions, plus one cached "eligible" bool that the engine recomputes from the 10 flags.

The 10 condition fields, in order:

1. `bGrowthFinished` (set when growth hits 1.0)
2. `bGrewUpInDifferentLocations` (visiting multiple zones during growth)
3. `bSocialized` (group activities with other players)
4. `bDrankFromMultipleSources` (varied water sources)
5. `bAteVariety` (varied diet)
6. `bDidNotInteractWithHumans` (no human contact during growth, on PvP variant only)
7. `bSurvived` (reaching adulthood alive)
8. `bRecovered` (recovering from injury, possibly health-related)
9. `bEarnedFavor` (some kind of favor-grant trigger)
10. `bUsedSpecialAbility` (using species-specific abilities)

The 11th field is the cached output bool, the one `GetIsEligiblePrimeElder()` returns to game code.

The engine's rule is roughly "if at least 5 of the 10 condition flags are true, set the cached eligible bool to true." The threshold check fires at specific gameplay moments: growth hitting 1.0, and some discrete events that re-evaluate prime eligibility.

Once eligible (5-of-10 threshold crossed), the cached bool gets locked in at the 75 percent growth mark. After that lock-in, the engine stops recomputing it from condition flags. This is what makes prime "stick" for a dino that earned it: the condition flags can later go stale or be overwritten, but the locked bool remains true.

For a dino that has not crossed the threshold, the cached bool is volatile. The engine recomputes it on every relevant tick. Setting the bool directly via `ServerSetPrimeEligible(true)` works for exactly one frame, then gets recomputed back to false from the 10 condition flags (most of which are false for a dino that hasn't actually earned prime).

## The naive approach that does not work

Early state-restore versions tried `pawn:ServerSetPrimeEligible(true)` to grant prime status. The bool flipped to true, then about a tick later flipped back to false. The restored dino would show prime briefly then lose it.

The interpretation that this was "the bool is read-only" or "the function is broken" was wrong. The actual behavior is "the engine recomputes the cached bool from the 10 condition flags, so setting the cached bool without also setting the condition flags is meaningless."

## The recipe that works

Force all 10 condition flags to true, force the cached eligible bool to true, and push the full struct via `SetEligiblePrimeElderData`. The engine sees all 10 flags as set (well above the 5-of-10 threshold), keeps the cached eligible bool true, and if the dino is at or past 75 percent growth, locks it in.

```lua
-- Force prime status by setting the full FEligiblePrimeElder struct.
-- The naive ServerSetPrimeEligible(true) does NOT stick because the engine
-- recomputes the cached eligible bool from the 10 condition flags. Forcing
-- all 10 flags AND the cached bool together produces a state that the engine
-- accepts and locks in (at or past 75% growth).

local function forcePrime(pawn)
    if pawn == nil then return false, "no-pawn" end

    -- Get the live struct. POD fields, safe to access by name.
    local pe = pawn:GetEligiblePrimeElderData()

    -- Set all 10 condition flags. The exact field names below match the
    -- struct definition in the UHT dump.
    pe.bGrowthFinished = true
    pe.bGrewUpInDifferentLocations = true
    pe.bSocialized = true
    pe.bDrankFromMultipleSources = true
    pe.bAteVariety = true
    pe.bDidNotInteractWithHumans = true
    pe.bSurvived = true
    pe.bRecovered = true
    pe.bEarnedFavor = true
    pe.bUsedSpecialAbility = true

    -- Set the cached output bool.
    pe.bIsEligiblePrimeElder = true

    -- Push the struct. No bForceReplication flag for this setter; the engine
    -- handles replication via the standard property system.
    local ok = pcall(function() pawn:SetEligiblePrimeElderData(pe) end)
    return ok
end
```

The `bDidNotInteractWithHumans` flag is interesting because on PvP-config servers this flag's meaning is roughly "did not encounter players during growth," which obviously is false for any player-controlled dino. Setting it to true anyway works; the engine's threshold check is just "at least 5 of these are true," it doesn't enforce any logical relationship between which 5.

## Capture for round-trip

For a state-restore mod, capture all 11 fields plus the cached bool. Restore via the recipe above. The capture is just field reads on the live struct, all POD.

```lua
local pe = pawn:GetEligiblePrimeElderData()
state.prime = {
    isPrime = pe.bIsEligiblePrimeElder,
    growthFinished = pe.bGrowthFinished,
    multipleLocations = pe.bGrewUpInDifferentLocations,
    socialized = pe.bSocialized,
    drankFromMultipleSources = pe.bDrankFromMultipleSources,
    ateVariety = pe.bAteVariety,
    didNotInteractWithHumans = pe.bDidNotInteractWithHumans,
    survived = pe.bSurvived,
    recovered = pe.bRecovered,
    earnedFavor = pe.bEarnedFavor,
    usedSpecialAbility = pe.bUsedSpecialAbility,
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

The mistake to avoid is treating `bIsEligiblePrimeElder` as the source of truth. It is a cached output, recomputed from the 10 condition flags. Setting it alone is undone the next time the engine recomputes.

If your mod's purpose is to grant prime as a reward (quest completion, admin grant, paid perk), use the force-prime recipe. It is the only path that survives the engine's recomputation logic across all growth levels.

If your mod's purpose is to capture and restore prime state without changing it, capture the full struct including all 10 condition flags, restore the full struct via `SetEligiblePrimeElderData`. The engine handles the rest.
