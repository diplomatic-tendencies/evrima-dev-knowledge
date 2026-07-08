# BodyDrop architecture

BodyDrop is an AI-free dead-body spawner for The Isle EVRIMA. It spawns a fresh corpse of the requested species at a chosen location, with the corpse's growth/weight matched to whatever the caller specified. The corpse is edible by other dinos and follows the engine's normal decay rules.

## The recipe

The pattern that produces a ragdolled, eatable corpse (without ever having an AI controller possess it). Order matters — `ActivateDeadbody` alone does not kill; it is post-death cleanup that expects `bIsDead=true` already, and `SetHealth(0)` alone leaves the pawn in idle animation:

```lua
local function spawnCorpse(speciesClassPath, location, growthFraction, ragdollSeconds)
    -- Step 1: resolve the pawn class
    local pawnCls = StaticFindObject(speciesClassPath)
    if pawnCls == nil then return nil, "class-not-found" end

    -- Step 2: get the world from gameMode
    local gm = findGameMode()
    if gm == nil then return nil, "no-game-mode" end
    local world = gm:GetWorld()
    if world == nil then return nil, "no-world" end

    -- Step 3: find REAL ground under the requested XY and spawn just above it.
    -- Do NOT spawn at a blind vertical offset like +1500: SpawnActor SUCCEEDS
    -- in the empty void below the landscape, which is exactly how corpses end
    -- up under the map. Trace straight down from high above the point to the
    -- true surface, then reject water separately (a corpse must not float or
    -- sink), spawn a little above the hit, and let the ragdoll (Step 10,
    -- ToggleServerRagdoll) settle it.
    local groundZ = traceGround(world, location.X, location.Y)  -- SphereTrace/LineTrace from a ceiling Z; nil if no solid ground
    if groundZ == nil then return nil, "no-ground" end
    if isPointWater(location.X, location.Y, groundZ) then return nil, "water" end
    local loc = { X = location.X, Y = location.Y, Z = groundZ + 300 }
    local rot = { Pitch = 0, Yaw = 0, Roll = 0 }
    local pawn
    local ok = pcall(function() pawn = world:SpawnActor(pawnCls, loc, rot) end)
    if not ok or pawn == nil then return nil, "spawn-failed" end

    -- Step 4: validate the nullptr wrapper (terrain collision returns a
    -- non-nil Lua wrapper around a null UObject; calling methods on it
    -- crashes the server). On failure, RESAMPLE a fresh XY around the anchor
    -- and re-trace ground — never retry with an untraced vertical/lateral
    -- offset (that is what put corpses under the map). Production BodyDrop
    -- resamples a handful of annulus positions before giving up.
    local addr
    pcall(function() addr = pawn:GetAddress() end)
    if addr == nil or addr == 0 then return nil, "nullptr-wrapper" end

    -- Step 5: replication. SetReplicates + the ForceNetUpdate at the end are all a
    -- corpse needs — it uses native distance relevancy exactly like a real
    -- death body. Do NOT set bAlwaysRelevant: an always-relevant corpse enters
    -- EVERY joining client's initial replication burst, and at scale that burst
    -- is a client load-in crash (this flag was removed from BodyDrop in v004
    -- for exactly that reason). bIsDead is a RepNotify, so clients entering
    -- cull range pose the corpse via OnRep_IsNowDead on their own.
    pcall(function() pawn:SetReplicates(true) end)

    -- Step 6: set growth BEFORE transitioning to corpse. Growth drives
    -- both the corpse mesh size (juvenile vs adult) and the food value.
    pcall(function() pawn:SetGrowth(growthFraction or 1.0) end)

    -- Step 7-11: full corpse-state sequence — ORDER MATTERS.
    -- Skipping any of these leaves the pawn in idle animation (not
    -- ragdolled) or animated (no death pose). All steps are needed.
    pcall(function() pawn:SetHealth(0) end)
    pcall(function() pawn.bIsDead = true end)             -- replicated death flag
    pcall(function() pawn:OnRep_IsNowDead() end)          -- trigger the OnRep callback manually
    pcall(function() pawn:ToggleServerRagdoll(true) end)  -- physical ragdoll
    pcall(function() pawn:ActivateDeadbody(false, ragdollSeconds or 3600) end)
                                                          -- corpse activation + forced-cleanup timer

    pcall(function() pawn:ForceNetUpdate() end)

    return pawn
end
```

The engine does NOT correctly produce a ragdolled corpse from just `SetHealth(0)`. Without `bIsDead`, `OnRep_IsNowDead`, `ToggleServerRagdoll`, and `ActivateDeadbody`, the pawn either stays in idle animation or T-poses. `ActivateDeadbody` is post-death cleanup; it expects the death state flags to be set already.

Two placement/replication rules the current mod (v004) settled the hard way, both encoded above and worth calling out because early BodyDrop got them wrong:

- **Ground-trace, never a blind vertical offset.** An early version spawned at `location.Z + 1500` and trusted ragdoll physics to drop the body to the surface. `SpawnActor` succeeds in the empty space *below* the landscape too, so an anchor point that was itself under-map (or a trace-free offset over a steep slope) produced corpses buried under the world. The fix is to trace down to real ground at the candidate XY, reject water, and spawn just above the hit.
- **No `bAlwaysRelevant`.** It looks like the flag that "makes clients see the corpse," but a corpse replicates fine on native distance relevancy (it is a dead pawn like any other, and `bIsDead` is a RepNotify). Marking corpses always-relevant instead forces every one of them into each joining client's initial replication burst; at a realistic corpse count that burst is a client load-in crash. v004 removed it.

The growth value drives both the corpse mesh size (juvenile vs adult) and the food value of the corpse. Setting growth to 1.0 produces an adult-sized corpse with full food value. Setting growth to 0.5 produces a juvenile-sized corpse with proportionally less food.

The `ragdollSeconds` parameter to `ActivateDeadbody` is the forced-cleanup timer. The game's natural decay (Fresh → NonFresh → Carcass → Rotten → Bones, per `ECorpseType`) handles intermediate state transitions; this timer is the upper bound. 3600 (1 hour) is the BodyDrop default.

## Species class paths

BodyDrop's verified species catalog. All under `/Game/TheIsle/Core/Characters/Dinosaurs/<Species>/BP_<Species>.BP_<Species>_C`:

| Species | Class path |
|---|---|
| Allosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Allosaurus/BP_Allosaurus.BP_Allosaurus_C` |
| Beipiaosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Beipiaosaurus/BP_Beipiaosaurus.BP_Beipiaosaurus_C` |
| Carnotaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Carnotaurus/BP_Carnotaurus.BP_Carnotaurus_C` |
| Ceratosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Ceratosaurus/BP_Ceratosaurus.BP_Ceratosaurus_C` |
| Compsognathus | `/Game/TheIsle/Core/Characters/Dinosaurs/Compsognathus/BP_Compsognathus.BP_Compsognathus_C` |
| Deinosuchus | `/Game/TheIsle/Core/Characters/Dinosaurs/Deinosuchus/BP_Deinosuchus.BP_Deinosuchus_C` |
| Diabloceratops | `/Game/TheIsle/Core/Characters/Dinosaurs/Diabloceratops/BP_Diabloceratops.BP_Diabloceratops_C` |
| Dilophosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Dilophosaurus/BP_Dilophosaurus.BP_Dilophosaurus_C` |
| Dryosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Dryosaurus/BP_Dryosaurus.BP_Dryosaurus_C` |
| Gallimimus | `/Game/TheIsle/Core/Characters/Dinosaurs/Gallimimus/BP_Gallimimus.BP_Gallimimus_C` |
| Herrerasaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Herrerasaurus/BP_Herrerasaurus.BP_Herrerasaurus_C` |
| Hypsilophodon | `/Game/TheIsle/Core/Characters/Dinosaurs/Hypsilophodon/BP_Hypsilophodon.BP_Hypsilophodon_C` |
| Maiasaura | `/Game/TheIsle/Core/Characters/Dinosaurs/Maiasaura/BP_Maiasaura.BP_Maiasaura_C` |
| Omniraptor | `/Game/TheIsle/Core/Characters/Dinosaurs/Omniraptor/BP_Omniraptor.BP_Omniraptor_C` |
| Pachycephalosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Pachycephalosaurus/BP_Pachycephalosaurus.BP_Pachycephalosaurus_C` |
| Pteranodon | `/Game/TheIsle/Core/Characters/Dinosaurs/Pteranodon/BP_Pteranodon.BP_Pteranodon_C` |
| Stegosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Stegosaurus/BP_Stegosaurus.BP_Stegosaurus_C` |
| Tenontosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Tenontosaurus/BP_Tenontosaurus.BP_Tenontosaurus_C` |
| Triceratops | `/Game/TheIsle/Core/Characters/Dinosaurs/Triceratops/BP_Triceratops.BP_Triceratops_C` |
| Troodon | `/Game/TheIsle/Core/Characters/Dinosaurs/Troodon/BP_Troodon.BP_Troodon_C` |
| Tyrannosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Tyrannosaurus/BP_Tyrannosaurus.BP_Tyrannosaurus_C` |

Other classes (Pterodactylus, Psittacosaurus biome variants) also spawn — see `EVRIMA_AI_Spawn_Pairs.md` for the full pawn catalog. BodyDrop's table above is the subset the mod ships with as corpse options.

Animal species (Deer, Boar, Goat, etc.) also work with the same recipe using paths under `/Game/TheIsle/Core/Characters/Animals/`. The full animal class list is in `EVRIMA_Spawnable_Actors.md`.

The visual is identical to a real-player-killed corpse of the same species. The food value matches what an adult of that species would yield. Players can eat these corpses normally.

## Filter system (anchor-eligibility)

BodyDrop v002 adds an anchor-eligibility filter system. "Anchors" are the live players the spawn loop radiates corpses around. The filters gate WHICH players the spawn loop services, not WHO can call the commands.

Real config knobs (defaults from `BodyDrop\Scripts\main.lua`; many production deployments override via `Saved/config.json`):

| Knob | Source default | Effect |
|---|---|---|
| `growthFilterEnabled` | false | When true, skip anchors above `maxGrowthPercent` |
| `maxGrowthPercent` | 70 | Threshold for the growth filter (anchors must be smaller than this to qualify) |
| `hungerFilterEnabled` | false | When true, skip anchors whose hunger is below `maxHungerPercent` (i.e. only spawn for hungry players) |
| `maxHungerPercent` | 15 | Hunger threshold (in HUD-percent terms) |
| `spawnForHerbivores` | true | When false, herbivore anchors are skipped |
| `excludeSameSpeciesAsAnchor` | false | When true, the spawned corpse won't be the same species as the anchor player |
| `groupPolicy` | `"individual"` | One of `individual` / `per_group` / `solo_only` — see GroupId-aware spawning |

The filters apply per-tick; each tick the spawn loop builds the eligible-anchor list, picks an anchor, picks a species (excluding the anchor's species if configured), and spawns one corpse at a scatter offset around that anchor.

There is no per-caller rate limit on the inbox / chat command surface itself — the rate-limiting is the spawn loop's own tick interval.

## Performance notes

A single corpse spawn takes about 20 to 40 milliseconds on the test rig. Most of that is the SpawnActor call itself; the SetHealth and replication flag updates are sub-millisecond.

The engine's corpse decay reclaims the actor automatically. No mod-side cleanup is needed. This is the key reason BodyDrop is safe and most other actor-spawning patterns are not: BodyDrop never holds references to spawned actors past the spawn call, so there's no chance of touching a freed actor (safety rule 9b).

## Other mod surfaces

In addition to the spawn-loop + recipe primitive, BodyDrop exposes:

- **Chat command surface**: `!bd` / `!bodydrop` with subcommands like `diag`, `status`, `set interval N`, `set <filter> <on/off>`, etc.
- **CommandBridge inbox handler**: tails `Mods/BodyDrop/Saved/inbox.ndjson` accepting the same token-array commands. Routed via CommandBridge's `bd`/`bodydrop` verb.
- **NDJSON events emitter**: `Mods/BodyDrop/Saved/events.ndjson` for downstream tracking of every spawned corpse (steam of anchor, species, location, ts).
- **Group-aware spawning**: when `groupPolicy=per_group`, the spawn loop counts each `TICharacterBase.GroupId` only once per tick (so a group of 4 friends gets 1 corpse allocation, not 4); `solo_only` excludes any player who shares a GroupId with another connected player.

See the CommandBridge architecture doc for how the inbox routing wires up.

## Closing notes

The recipe above is everything required for the spawn primitive. There is no extra setup, no controller construction, no possession. The engine handles the corpse lifecycle **once all the corpse-state flags are written in the right order** — `SetHealth(0)` alone is not enough.

If your mod uses this pattern, do not store the returned pawn pointer for later use. The engine's corpse decay will destroy it on its own schedule. Lua-side cleanup via `K2_DestroyActor` on a stored pointer will crash the server (safety rule 9b).
