# EVRIMA per-species group size

The maximum size of a group (herd or pack) is a per-species value the server controls, and you can change it from a mod. This is the mechanism behind a "set the herd cap for this species" feature, and it is short, with one wrinkle around persistence.

## The field

`MaxGroupSize` is an int inside `GeneralSettings` on `TICharacterBase`. `GeneralSettings` is the per-species settings block, so the cap is naturally per-species: a Carnotaurus pawn carries the Carnotaurus cap, a Maiasaura pawn the Maiasaura cap. It is plain old data — a direct int read and write, no marshaling concerns.

The getter `GetMaxGroupSize()` mirrors the field verbatim (verified live). Which of the two the native join gate actually consults I have not traced — a native call would bypass the reflected getter, so do not count on hooking `GetMaxGroupSize()` to observe or intercept the decision. For changing the cap it does not matter: the gate is server-authoritative and follows the field either way — writing it changes the limit the server actually enforces, verified by forming a 4-member group at a written cap of 6 on a species whose stock cap is 3.

```lua
-- read and set the cap on a live pawn
local before = pawn.GeneralSettings.MaxGroupSize
pawn.GeneralSettings.MaxGroupSize = newCap
local after  = pawn.GeneralSettings.MaxGroupSize       -- confirm the write took
local getter = pawn:GetMaxGroupSize()                  -- and that the getter agrees
```

One caveat that will make you think the write failed when it did not: `GeneralSettings` is not a replicated property, so the server-side write never reaches clients — the client's group UI can keep showing the stock cap while the server happily lets the group grow past it. Verify by actually forming the larger group, not by reading the number in the client UI.

## The persistence wrinkle

The write does not survive a respawn on its own. When a pawn initializes, the species default re-stamps `MaxGroupSize` (and likely the rest of `GeneralSettings`) back to the class value. So a one-time write holds for the current pawn but is gone the next time that player spawns.

The fix is a re-apply sweep: keep your desired per-species caps in config, and on a periodic tick walk the online pawns and re-write the cap on any whose value does not match your target. This is the same shape as any "keep a field stamped against engine re-initialization" pattern. Run it on a slow cadence (every few seconds is plenty; the cap only matters at the moment of a join attempt) and only write when the current value differs, so you are not churning the field every tick.

```lua
-- per-sweep, for each online pawn
local species = shortSpecies(pawn)            -- BP_<Species>_C leaf, normalized
local want = configCaps[species]
if want ~= nil and pawn.GeneralSettings.MaxGroupSize ~= want then
    pawn.GeneralSettings.MaxGroupSize = want
end
```

## Closing notes

One field, server-authoritative, per-species, re-stamped on spawn. Write it to change the cap, and re-apply on a sweep to make the change durable across respawns. That is the whole mechanism. The only thing to keep in mind is the re-stamp, which is easy to miss if you test the write once on a live pawn, see it hold, and assume it persists — it does not until you sweep.
