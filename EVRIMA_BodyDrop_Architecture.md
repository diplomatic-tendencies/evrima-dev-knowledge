# BodyDrop architecture

BodyDrop is an AI-free dead-body spawner for The Isle EVRIMA. It spawns a fresh corpse of the requested species at a chosen location, with the corpse's growth/weight matched to whatever the caller specified. The corpse is edible by other dinos and follows the engine's normal decay rules.

Use cases: admin food drops for hungry players, event rewards, lure points for AI hunters, scripted scenario setup.

The mod is intentionally narrow in scope. It does not spawn live AI dinos (that's `EVRIMA_AI_Spawn_Pairs.md`). It does not spawn world actors like trees or static meshes. Just corpses.

## The recipe

The six-step pattern that produces a corpse-as-spawned (without ever having an AI controller possess it):

```lua
local function spawnCorpse(speciesClassPath, location, growthFraction)
    -- Step 1: resolve the pawn class
    local pawnCls = StaticFindObject(speciesClassPath)
    if pawnCls == nil then return nil, "class-not-found" end

    -- Step 2: get the world from any pawn or gameMode
    local gm = findGameMode()
    if gm == nil then return nil, "no-game-mode" end
    local world = gm:GetWorld()
    if world == nil then return nil, "no-world" end

    -- Step 3: spawn the pawn
    local loc = { X = location.X, Y = location.Y, Z = location.Z + 500 }
    local rot = { Pitch = 0, Yaw = 0, Roll = 0 }
    local pawn
    local ok = pcall(function() pawn = world:SpawnActor(pawnCls, loc, rot) end)
    if not ok or pawn == nil then return nil, "spawn-failed" end

    -- Step 4: validate the nullptr wrapper (terrain collision returns one)
    local addr
    pcall(function() addr = pawn:GetAddress() end)
    if addr == nil or addr == 0 then return nil, "nullptr-wrapper" end

    -- Step 5: set growth (drives the corpse mesh size and food value)
    pcall(function() pawn:SetGrowth(growthFraction or 1.0) end)

    -- Step 6: kill it immediately
    pcall(function() pawn:SetHealth(0) end)

    -- Optional: enable replication so the client sees it
    pcall(function() pawn:SetReplicates(true) end)
    pcall(function() pawn.bAlwaysRelevant = true end)
    pcall(function() pawn:ForceNetUpdate() end)

    return pawn
end
```

The key insight is that the engine handles the "this dino is dead" flow correctly if you just spawn the pawn class and immediately set its Health to zero. There is no need to instantiate an AI controller, possess the pawn, then kill it. Skipping the controller saves the construction overhead and avoids the chance of an AI brain firing on a pawn that's about to die.

The growth value drives both the corpse mesh size (juvenile vs adult) and the food value of the corpse. Setting growth to 1.0 produces an adult-sized corpse with full food value. Setting growth to 0.5 produces a juvenile-sized corpse with proportionally less food.

## Species class paths

The 23 species class paths verified to spawn as corpses without crashing:

| Species | Class path |
|---|---|
| Tyrannosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Tyrannosaurus/BP_Tyrannosaurus.BP_Tyrannosaurus_C` |
| Carnotaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Carnotaurus/BP_Carnotaurus.BP_Carnotaurus_C` |
| Ceratosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Ceratosaurus/BP_Ceratosaurus.BP_Ceratosaurus_C` |
| Dilophosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Dilophosaurus/BP_Dilophosaurus.BP_Dilophosaurus_C` |
| Allosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Allosaurus/BP_Allosaurus.BP_Allosaurus_C` |
| Herrerasaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Herrerasaurus/BP_Herrerasaurus.BP_Herrerasaurus_C` |
| Omniraptor | `/Game/TheIsle/Core/Characters/Dinosaurs/Omniraptor/BP_Omniraptor.BP_Omniraptor_C` |
| Beipiaosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Beipiaosaurus/BP_Beipiaosaurus.BP_Beipiaosaurus_C` |
| Pteranodon | `/Game/TheIsle/Core/Characters/Dinosaurs/Pteranodon/BP_Pteranodon.BP_Pteranodon_C` |
| Pterodactylus | `/Game/TheIsle/Core/Characters/Dinosaurs/Pterodactylus/BP_Pterodactylus.BP_Pterodactylus_C` |
| Triceratops | `/Game/TheIsle/Core/Characters/Dinosaurs/Triceratops/BP_Triceratops.BP_Triceratops_C` |
| Diabloceratops | `/Game/TheIsle/Core/Characters/Dinosaurs/Diabloceratops/BP_Diabloceratops.BP_Diabloceratops_C` |
| Stegosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Stegosaurus/BP_Stegosaurus.BP_Stegosaurus_C` |
| Pachycephalosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Pachycephalosaurus/BP_Pachycephalosaurus.BP_Pachycephalosaurus_C` |
| Maiasaura | `/Game/TheIsle/Core/Characters/Dinosaurs/Maiasaura/BP_Maiasaura.BP_Maiasaura_C` |
| Tenontosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Tenontosaurus/BP_Tenontosaurus.BP_Tenontosaurus_C` |
| Hypsilophodon | `/Game/TheIsle/Core/Characters/Dinosaurs/Hypsilophodon/BP_Hypsilophodon.BP_Hypsilophodon_C` |
| Dryosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Dryosaurus/BP_Dryosaurus.BP_Dryosaurus_C` |
| Gallimimus | `/Game/TheIsle/Core/Characters/Dinosaurs/Gallimimus/BP_Gallimimus.BP_Gallimimus_C` |
| Compsognathus | `/Game/TheIsle/Core/Characters/Dinosaurs/Compsognathus/BP_Compsognathus.BP_Compsognathus_C` |
| Troodon | `/Game/TheIsle/Core/Characters/Dinosaurs/Troodon/BP_Troodon.BP_Troodon_C` |
| Psittacosaurus (Coastal) | `/Game/TheIsle/Core/Characters/Dinosaurs/Psittacosaurus/BP_Psittacosaurus_Coastal.BP_Psittacosaurus_Coastal_C` |
| Psittacosaurus (Highlands) | `/Game/TheIsle/Core/Characters/Dinosaurs/Psittacosaurus/BP_Psittacosaurus.BP_Psittacosaurus_C` |

Animal species (Deer, Boar, Goat, etc.) also work, with the same recipe, using paths under `/Game/TheIsle/Core/Characters/Animals/`. The full animal class list is in `EVRIMA_Spawnable_Actors.md`.

## Cut-content species

Triceratops, Stegosaurus, Diabloceratops, Pachycephalosaurus, Maiasaura, and a few others are in the game's class registry but not in the live spawn rotation as playable species. They still spawn fine as corpses; the engine has no problem materializing the pawn class, killing it, and having the corpse mesh render.

The visual is identical to a real-player-killed corpse of the same species. The food value matches what an adult of that species would yield. Players can eat these corpses normally.

The exception is Kentrosaurus and Parasaurolophus. Both have BP class entries but the classes are stubs without full mesh and animation data. Spawning them produces an invisible corpse that gives food but has no visual. Skip these unless you have a specific use for an invisible food source.

## Filter system

BodyDrop v002 adds a filter system to gate spawns by admin permission and prevent spam. The filter logic runs before the spawn:

```lua
local function checkSpawnAllowed(callerSteam, species, growth)
    if not isAdminOrAllowed(callerSteam) then
        return false, "not-allowed"
    end
    if rateLimit[callerSteam] and (os.time() - rateLimit[callerSteam]) < 5 then
        return false, "rate-limit"
    end
    if growth < 0.1 or growth > 1.0 then
        return false, "growth-out-of-range"
    end
    if not isValidSpecies(species) then
        return false, "unknown-species"
    end
    return true
end
```

The rate limit is per-caller, not global. The growth range prevents hatchling-sized corpse spam (which would still give some food, just less). The species validation rejects typos before they reach the spawn API.

## Performance notes

A single corpse spawn takes about 20 to 40 milliseconds on the test rig. Most of that is the SpawnActor call itself; the SetHealth and replication flag updates are sub-millisecond.

The engine's corpse decay (about 5 minutes for adult dinos at default settings) reclaims the actor automatically. No mod-side cleanup is needed. This is the key reason BodyDrop is safe and most other actor-spawning patterns are not: BodyDrop never holds references to spawned actors past the spawn call, so there's no chance of touching a freed actor (safety rule 9b).

## Closing notes

The 6-step recipe in this document is everything required. There is no extra setup, no controller construction, no possession. The engine handles the rest.

If your mod uses this pattern, do not store the returned pawn pointer for later use. The engine's corpse decay will destroy it on its own schedule. Lua-side cleanup via `K2_DestroyActor` on a stored pointer will crash the server (safety rule 9b).
