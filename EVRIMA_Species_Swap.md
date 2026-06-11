# EVRIMA live species swap

This is the working recipe for swapping a player's dino to a different species **live**  no kill, no respawn screen, no client-side anything. The player types a command and is standing in the world as a different species two seconds later, with movement, abilities, HUD, admin tooling, and save persistence all intact.

The naive version of this feature is one call: `controller:Possess(newPawn)`. That call alone *appears* to work and silently breaks three engine systems. Most of this document is about the three bindings you must also write, how each was discovered, and the one function that shuts down your server.

## Why the obvious paths fail

| Path | Result |
|---|---|
| `RequestRespawn(...)` | Crashes from Lua, always — the by-value `FCustomizerDataBase` parameter contains an FString (safety rule 4). |
| `RequestSpawnInNestAsSpecies(class, steam)` | Callable, but requires the player to be on the respawn screen and spawns at an owned nest. Not a live swap. |
| `RequestDevSpawn(class)` | **SHUTS DOWN THE SERVER.** See the warning below. |
| `ctrl:Possess(newPawn)` alone | Works at the controller level; engine bookkeeping breaks (details below). |

### RequestDevSpawn is a server killer

`TIPlayerController:RequestDevSpawn(TSubclassOf<ATICharacterBase>)` looks like exactly what you want  a one-parameter dev spawn. Calling it server-side from Lua triggers a **clean engine shutdown**: orderly module teardown, `LogExit: Exiting`, no crash dump, server gone within seconds. Verified the hard way. It is presumably a client-RPC entry that hits a fatal path when invoked directly on a dedicated server. Never call it.

## The mechanism: possession plus three bindings

A spawned pawn that a player controller possesses is, to the engine, three different things — and raw `Possess` only delivers the first:

1. **The controlled pawn** (controller level). `Possess` handles this. The client's camera, input mapping, movement mode, and ability inputs all rewire automatically through the standard UE possession flow verified for walking, swimming, and flying pawns, with zero client-side fixups.

2. **The player's character** (ownership level). `ATICharacterBase` carries `FString SteamId` (offset 0x0AA8 on this build) with a UFunction setter `SetSteamId(FString)`. This field is how the engine answers "whose dino is this": `gm:GetCharacterToRepossess(steam)` — the reconnect-rebind lookup — matches on it. A possessed pawn with an empty SteamId is nobody's character: reconnect-repossess breaks and the save system loses track. FString *parameters* marshal safely from Lua (same shape as `GetControllerBySteamId`); this is not a rule-2 violation.

3. **A registered player character** (admin/RCON level). `TIGameModeBase.AllPlayerCharacters` is the array the natural spawn flow appends to, and it is what RCON `getPlayerData` (opcode `0x77`) iterates. A possessed, SteamId-bound pawn that isn't in this array is invisible to the entire admin data layer — `getPlayerData` returns an empty list for that player while they happily walk around. Verified by appending the pawn from Lua and watching `getPlayerData` flip from empty to a full correct row. **Append-only**: never read or call methods on existing entries (safety rule 11 — the array retains stale ghosts); the engine filters dead entries out of RCON output itself.

Two more pieces complete the illusion:

4. **The client HUD does not follow possession.** The natural spawn pipeline has an explicit add-HUD step (`bAddHud` parameter on `RequestRespawn`); raw possession skips it, so the player's vital bars stay frozen on the old pawn's attribute set. Server-side writes land fine — admin heal/set-hunger commands *look* broken while working perfectly. The fix is one call on the controller after possession: `RequestOnRespawnHudUpdate()`. Player-confirmed: bars come back to life immediately.

5. **Persistence** is the character's own job: `pawn:SaveDataToFile(bool bWasSafeLog)` writes the player's `.sav`. Verified by relogging — the engine greeted the player with `Save file found Dino: BP_Triceratops_C` and loaded the swapped species through its completely normal path. (i pass `false`; the semantics of `true` vs `false` beyond this are untested.)

## The recipe (all steps verified live)

```lua
-- speciesCls = StaticFindObject("/Game/.../BP_<Species>.BP_<Species>_C")
-- ctrl       = gm:GetControllerBySteamId(steam)   (fresh, never cached)
-- oldPawn    = livePawnFromCtrl(ctrl)             (rule-9a address check)

-- 1. Spawn the new species near the player. Offset ladder beats terrain
--    collision; validate the nullptr wrapper (rule 9a).
local newPawn = world:SpawnActor(speciesCls,
    { X = loc.X, Y = loc.Y, Z = loc.Z + 300 }, { Pitch = 0, Yaw = yaw, Roll = 0 })
-- check newPawn:GetAddress() ~= 0; retry with bigger offsets if not

-- 2. Network flags + state before anyone sees it
newPawn:SetReplicates(true)
newPawn.bAlwaysRelevant = true
newPawn:SetGrowth(targetGrowth)
newPawn:SetHealth(newPawn:GetMaxHealth())

-- 3. Ownership binding (the reconnect/save identity)
newPawn:SetSteamId(steam)

-- 4. Admin/RCON registration (append-only; never touch existing entries)
local arr = gm.AllPlayerCharacters
arr[#arr + 1] = newPawn

-- 5. The possession itself
ctrl:Possess(newPawn)

-- 6. Verify synchronously: possession lands in the same tick server-side
local nowPawn = livePawnFromCtrl(ctrl)
-- compare nowPawn:GetAddress() with newPawn's address

-- 7. Retire the old body IN THE SAME TICK (see disposal section)
oldPawn:SetSteamId("")          -- exactly one character per steam
oldPawn:K2_DestroyActor()

-- 8. Rebind the client HUD + clear any stuck ability-block tag
ctrl:RequestOnRespawnHudUpdate()
newPawn:VerifyAndRemoveBlockAbilitiesTag()

-- 9. Apply whatever state you're restoring (growth, vitals, mutations,
--    skin — the standard state-restore cookbook applies unchanged), then:
newPawn:SaveDataToFile(false)
```

In production, wrap step 6 in a deferred re-verify one tick later (~2s) as a fallback, and treat the synchronous check as the fast path. Every call above is pcall-wrapped in the real implementation; shown bare for readability.

## Disposing of the old body

Corpse-ifying the old pawn (the BodyDrop recipe) works but leaves an **eatable body** — store-swap-eat-your-own-corpse is a free-meat dupe, and the lingering corpse ruins the transformation illusion. The production answer is `K2_DestroyActor` — which safety rule 9b ordinarily bans.

The ban is about *stale* wrappers: every observed 9b crash was a destroy on an actor that gameplay had already destroyed. This is the one situation where the preconditions for a safe destroy genuinely hold:

- the wrapper is **same-tick fresh** (obtained in this function call),
- the actor is **known alive** (it was the player's live pawn milliseconds ago),
- it is **exclusively ours** (unpossessed; nothing in the engine destroys an idle live dino in this window).

Confirm possession synchronously first, then destroy in the same tick. Multiple swaps verified live with no delayed AV. If you cannot satisfy all three conditions (e.g. your verify is deferred and the wrapper is seconds old), fall back to the corpse recipe with a short `ActivateDeadbody` timer and accept the brief body.

Destroy the *new* pawn by the same logic on every failure path (possession failed, verify failed) — no path should leak an orphan pawn into the world.

## Production hardening

**Race lock.** The spawn→possess→verify window is ~2 seconds. Two overlapping swap commands from the same player interleave catastrophically (two spawned pawns, possession ping-pong, a verify that retires the wrong body). Chat dedup does not save you — two *different* commands race fine. Keep a per-steam in-flight flag with a short self-expiry (I use 6s) and reject swaps while one is pending.

**Loss-proofing.** Auto-capture the outgoing dino's full state into a reserved slot before anything irreversible happens. A swap can then never lose a dino — the reserved slot is the undo button, and ping-ponging between two dinos via that slot falls out for free.

**Failure messaging.** Return "your original dino is untouched" on every abort path and mean it: nothing irreversible may happen before possession is confirmed.

## Known limitations and non-issues

**Gender does not carry over.** There is no `SetIsFemale` path (same limitation as transform-in-place restores); the spawned pawn's sex is spawn-default. A female player's dino can come back male. Tell your players.

**Short food lists on dinos are correct.** burned an hour on a "bug" where a swapped dinos preferred-food tab showed only three entries — a naturally-spawned dino of the same species shows the same three. some dino diet lists are just short. Server-side diet sets on swapped pawns were verified full the whole time (`DietSourceNamesByNutrient` per nutrient). Don't chase this one.

**Death deletes saves — by design.** If a player dies and relogs, "Save file not found" is normal engine behavior, not a swap regression. false-alarmed on this too.

## What this unlocks

The immediate feature is obvious: stored-dino redemption without the kill→respawn-as-the-same-species→redeem dance. Our implementation routes the classic `!redeem` through the swap engine behind a config toggle — same species takes the legacy transform-in-place, different species swaps — so players redeem from *any* current dino, including a fresh hatchling of the wrong species.

Less obviously, the bindings are the general-purpose answer to "how do I make the engine accept a Lua-spawned pawn as a real player character," which is the foundation for any feature in that space: species trials, shapeshifter events, admin-driven transformations, class-change shops. The possession mechanics were already proven for AI (see the AI spawn pair catalog); this document is the missing player-controller half.

## Verification summary

| Aspect | Status |
|---|---|
| `Possess` from Lua on a player controller | verified, crash-free, many reps |
| Camera/input/movement rewire (walk, swim, **fly**) | verified live, automatic |
| Species-specific abilities on swapped pawn (sniff, spear-fishing) | verified live |
| Admin gameplay commands target the swapped pawn | verified (failures were the frozen HUD illusion) |
| HUD rebind via `RequestOnRespawnHudUpdate` | verified, player-confirmed |
| `GetCharacterToRepossess` matches via SteamId | verified (offline-testable: tag a spawned pawn, query) |
| RCON `getPlayerData` via `AllPlayerCharacters` append | verified empty→populated |
| Relog persistence via `SaveDataToFile(false)` | verified ("Save file found" with swapped species) |
| Same-tick `K2_DestroyActor` of the old body | verified, multiple reps, no delayed AV |
| Gender carry-over | not possible (no setter) |
| `RequestDevSpawn` server-side | **engine shutdown — never call** |

## Closing notes

The lesson that generalizes: in EVRIMA, "the player's dino" is not one pointer, it's a *consensus* between the controller (possession), the character (SteamId), the game mode (AllPlayerCharacters), the client (HUD bindings), and the save system. The natural spawn pipeline builds that consensus in one place; a live swap has to rebuild it piece by piece, and every piece you skip fails somewhere different — possession without SteamId breaks reconnects, both without registration blinds your admin tools, all three without the HUD call gaslights the player into thinking your server is broken.

Each binding was found by watching a different observer disagree: UE4SS-side reads said "swapped" while RCON said "no such player" while the player's own HUD said "nothing changed." If you build anything in this space, set up all three viewpoints before you start — the disagreements are the diagnostic.
