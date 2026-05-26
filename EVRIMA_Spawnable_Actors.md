# EVRIMA spawnable world actors

This is a catalog of what spawns cleanly via `world:SpawnActor` server-side on The Isle EVRIMA, what doesn't, and why. The dinosaur and animal AI pairs are covered separately in `EVRIMA_AI_Spawn_Pairs.md`; this doc is everything else (cut-content species, VFX, fish, edible plants, gore items, world props).

## Cut-content dinosaur species

These have BP class entries in the game and spawn fine, but are not in the live spawn rotation as playable species. They are useful as set dressing, food sources (for BodyDrop), or AI fauna (with the cross-controller pairings in `EVRIMA_AI_Spawn_Pairs.md`).

| Species | Class path | Status |
|---|---|---|
| Triceratops | `/Game/TheIsle/Core/Characters/Dinosaurs/Triceratops/BP_Triceratops.BP_Triceratops_C` | Full mesh, no AI controller |
| Stegosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Stegosaurus/BP_Stegosaurus.BP_Stegosaurus_C` | Full mesh, no AI controller |
| Diabloceratops | `/Game/TheIsle/Core/Characters/Dinosaurs/Diabloceratops/BP_Diabloceratops.BP_Diabloceratops_C` | Full mesh, has AI controller |
| Maiasaura | `/Game/TheIsle/Core/Characters/Dinosaurs/Maiasaura/BP_Maiasaura.BP_Maiasaura_C` | Full mesh, no AI controller |
| Tenontosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Tenontosaurus/BP_Tenontosaurus.BP_Tenontosaurus_C` | Full mesh, has AI controller |
| Pachycephalosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Pachycephalosaurus/BP_Pachycephalosaurus.BP_Pachycephalosaurus_C` | Full mesh, no AI controller |
| Allosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Allosaurus/BP_Allosaurus.BP_Allosaurus_C` | Full mesh, has AI controller |
| Dryosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Dryosaurus/BP_Dryosaurus.BP_Dryosaurus_C` | Full mesh, has AI controller |
| Beipiaosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Beipiaosaurus/BP_Beipiaosaurus.BP_Beipiaosaurus_C` | Full mesh, no AI controller |
| Troodon | `/Game/TheIsle/Core/Characters/Dinosaurs/Troodon/BP_Troodon.BP_Troodon_C` | Full mesh, no AI controller |

The "no AI controller" entries can be paired with a similar-physiology controller from another species. Triceratops driven by Diabloceratops AI works (the behavior tree uses TICharacterBase APIs that all dinos share). Cross-controller pairings are in `EVRIMA_AI_Spawn_Pairs.md`.

## Stub species (avoid)

| Species | Class path | Why to avoid |
|---|---|---|
| Kentrosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Kentrosaurus/BP_Kentrosaurus.BP_Kentrosaurus_C` | Stub class without full mesh and animation data. Spawns invisible. |
| Parasaurolophus | `/Game/TheIsle/Core/Characters/Dinosaurs/Parasaurolophus/BP_Parasaurolophus.BP_Parasaurolophus_C` | Same as above. Stub class. |

These spawn cleanly (no crash), but the resulting actor is invisible and partially functional. Skip them unless you have a specific use for an invisible food source (a corpse for BodyDrop that yields food without rendering).

## Animals

The animal classes spawn cleanly. Every species below has at least one working AI controller (verified live by ControllerProbe 2026-05-22) — see `EVRIMA_AI_Spawn_Pairs.md` for the full pairing catalog with movement deltas.

| Species | Pawn class path | AI controller (one option) |
|---|---|---|
| Boar | `/Game/TheIsle/Core/Characters/Animals/Boar/BP_Boar.BP_Boar_C` | `/Script/TheIsle.TIAIBoarController` or BP wrapper |
| Boar (baby) | `/Game/TheIsle/Core/Characters/Animals/Boar/BP_Boar_Baby.BP_Boar_Baby_C` | Same controller as adult Boar |
| Chicken | `/Game/TheIsle/Core/Characters/Animals/Chicken/BP_Chicken.BP_Chicken_C` | `/Script/TheIsle.TIAIChickenController` or BP wrapper |
| Crab | `/Game/TheIsle/Core/Characters/Animals/Crab/BP_Crab.BP_Crab_C` | `/Script/TheIsle.TIAICrabController` or BP wrapper |
| Deer | `/Game/TheIsle/Core/Characters/Animals/Deer/BP_Deer.BP_Deer_C` | `/Game/.../BP_AI_Deer_Controller.BP_AI_Deer_Controller_C` |
| Deer (baby) | `/Game/TheIsle/Core/Characters/Animals/Deer/BP_Deer_Baby.BP_Deer_Baby_C` | `/Game/.../BP_AI_Deer_Baby_Controller.BP_AI_Deer_Baby_Controller_C` |
| Goat | `/Game/TheIsle/Core/Characters/Animals/Goat/BP_goat.BP_Goat_C` | `/Script/TheIsle.TIAIGoatController` or BP wrapper |
| Goat (baby) | `/Game/TheIsle/Core/Characters/Animals/Goat/BP_Goat_Baby.BP_Goat_Baby_C` | Same controller as adult Goat |
| Rabbit | `/Game/TheIsle/Core/Characters/Animals/Rabbit/BP_Rabbit.BP_Rabbit_C` | `/Script/TheIsle.TIAIRabbitController` or BP wrapper |
| Bullfrog | `/Game/TheIsle/Core/Characters/Animals/Bullfrog/BP_Bullfrog.BP_Bullfrog_C` | `/Script/TheIsle.TIAIFrogController` or BP wrapper |
| Seaturtle | `/Game/TheIsle/Core/Characters/Animals/Seaturtle/BP_Seaturtle.BP_Seaturtle_C` | `/Script/TheIsle.TIAISeaturtleController` or BP wrapper |

Note: the case in the actual class names matters. `BP_Seaturtle.BP_Seaturtle_C` (lowercase t) not `BP_SeaTurtle`. The pawn class for the frog is `BP_Bullfrog`, not `BP_Frog`. The pawn class for the crab is `BP_Crab`, not `BP_Crab_new` (that's the *AnimBlueprint*). The pawn class for the rabbit is `BP_Rabbit`, not `BP_RabbitNew`. See `EVRIMA_AI_Spawn_Pairs.md` for verified pairings.

## Fish

All under `/Game/TheIsle/Core/Characters/Fishes/` (note "Fishes", not "Fish"):

| Species | Class path | Notes |
|---|---|---|
| Elite Catfish | `/Game/TheIsle/Core/Characters/Fishes/Catfish/BP_Elite_Fish_CatFish.BP_Elite_Fish_CatFish_C` | Spawns; AI needs water context (NO_BRAIN if on land) |
| Elite Coelacanth | `/Game/TheIsle/Core/Characters/Fishes/Coelacanth/BP_Elite_Fish_Coelacanth.BP_Elite_Fish_Coelacanth_C` | Spawns; same water requirement |
| School Catfish | `/Game/TheIsle/Core/Characters/Fishes/Schools/BP_Fish_Catfish.BP_Fish_Catfish_C` | School fish |
| School Hoplosternum | `/Game/TheIsle/Core/Characters/Fishes/Schools/BP_Fish_Hoplosternum.BP_Fish_Hoplosternum_C` | School fish |
| School Longear | `/Game/TheIsle/Core/Characters/Fishes/Schools/BP_Fish_Longear.BP_Fish_Longear_C` | School fish |
| School MuskelLunge | `/Game/TheIsle/Core/Characters/Fishes/Schools/BP_Fish_MuskelLunge.BP_Fish_MuskelLunge_C` | School fish |
| School RainbowFish | `/Game/TheIsle/Core/Characters/Fishes/Schools/BP_Fish_RainbowFish.BP_Fish_RainbowFish_C` | School fish |

Fish AI is tied to water-presence checks. ControllerProbe verified that spawning a fish on dry land produces a possessed-but-idle pawn (`NO_BRAIN`, movement delta = 0). Spawn over a water area for working fish AI. No `Trout` or `Pike` class exists in the EVRIMA build.

## Edible plants

Various foliage classes are spawnable and provide food when eaten by herbivores:

| Plant | Class path |
|---|---|
| Fern | `/Game/TheIsle/Core/Foliage/Plants/BP_Fern.BP_Fern_C` |
| Cycad | `/Game/TheIsle/Core/Foliage/Plants/BP_Cycad.BP_Cycad_C` |
| Mushroom (edible) | `/Game/TheIsle/Core/Foliage/Plants/BP_Mushroom_Edible.BP_Mushroom_Edible_C` |

These spawn as static-positioned interactables. The herbivore "eat" interaction works on them.

## Gore items

| Item | Class path | Notes |
|---|---|---|
| GoreChunck (sic) | `/Game/TheIsle/Core/Characters/BP_GoreChunck.BP_GoreChunck_C` | Standalone meat chunk. Edible. |
| ItemPersistentData blob | Various | Engine-internal; not directly spawnable. |

The GoreChunck class is useful as a manual food drop without spawning a full corpse. Smaller and decays faster than a corpse.

## Effect actors

Visual effect actors mostly spawn cleanly on the server side, but only one renders correctly client-side. The others spawn server-side and replicate, but their visual is gated by the engine's effect-rendering system which the dedicated server does not run.

| Effect | Class path | Renders? |
|---|---|---|
| Halluc Fog | `/Game/TheIsle/Core/VFX/BP_HallucFog.BP_HallucFog_C` | YES |
| Halluc Char | `/Game/TheIsle/Core/Characters/BP_HallucinationChar.BP_HallucinationChar_C` | NO (server-side gate) |
| Smite Effect | `/Game/TheIsle/Core/VFX/BP_SmiteEffect.BP_SmiteEffect_C` | NO (server-side gate) |
| Various ambient FX | `/Game/TheIsle/Core/VFX/BP_*` | All NO |

The halluc fog is the exception that renders because it uses a fog volume actor whose visual data IS in the replicated state. Other VFX use Niagara particle systems that require the rendering pipeline to be active, which the dedicated server doesn't have.

The naive workaround of trying to spawn a raw Niagara particle component server-side does not work. The component spawns, replicates, and does nothing on the client. The component's particle system reference is not in the replicated state.

The path to custom visual effects on EVRIMA is `.pak` content (custom BP assets built in Unreal Editor with the desired Niagara baked into the BP) plus a server-side spawner. This is documented in `EVRIMA_Client_Mod_Feasibility.md`. The raw Niagara from Lua approach is a dead end.

## Skeleton meshes

Various skeleton-only mesh actors exist in the class registry. They spawn cleanly but their visual is again gated by client-side rendering. A skeleton mesh actor spawned server-side replicates, but the client receives an empty actor with no mesh.

The same root cause applies as VFX: server-side spawned actors don't carry their mesh references through replication unless the mesh is baked into the BP class default.

## Burrow construction kit

The game has a set of placeable burrow-segment actors used by the burrow-building system. Each spawns cleanly:

- `BP_Burrow_Entrance_C`
- `BP_Burrow_Segment_Straight_C`
- `BP_Burrow_Segment_Turn_C`
- `BP_Burrow_Room_C`

These are useful for manual burrow placement (admin commands, scripted scenarios). They behave like other spawnable BP actors.

## Nests

The nest classes are the most well-tested spawnable BPs:

- `BP_Nest_Mound_Large_H_C`
- `BP_Nest_Mound_Small_H_C`
- `BP_Nest_Burrow_H_C`
- `BP_Nest_Tree_H_C`
- And about 7 other variants

All 11 nest classes spawn cleanly when called via `world:SpawnActor(classObj, loc, rot)`. The naked spawn (without setting fields like `Code`, `MaleSteamId`) produces a nest in a default state. Setting the relevant FName/FString fields after spawn requires care due to FString marshaling restrictions (safety rule 2); the `Code` field as an FString is unsafe to read or write from Lua.

## Spawn primitive pattern (verified safe)

```lua
local function spawnWorldActor(classPath, loc, rot)
    local cls = StaticFindObject(classPath)
    if cls == nil then return nil, "class-not-found" end

    local gm = findGameMode()
    if gm == nil then return nil, "no-game-mode" end
    local world = gm:GetWorld()
    if world == nil then return nil, "no-world" end

    local actor
    local ok = pcall(function() actor = world:SpawnActor(cls, loc, rot) end)
    if not ok or actor == nil then return nil, "spawn-failed" end

    local addr
    pcall(function() addr = actor:GetAddress() end)
    if addr == nil or addr == 0 then return nil, "nullptr-wrapper" end

    -- Enable replication for client visibility
    pcall(function() actor:SetReplicates(true) end)
    pcall(function() actor.bAlwaysRelevant = true end)
    pcall(function() actor:ForceNetUpdate() end)

    return actor
end
```

This works for nests, burrow segments, edible plants, gore items, and most BP actors. It does not produce a visible mesh for AStaticMeshActor (rule 9; the mesh field doesn't replicate). It does not produce visual particles for Niagara-based VFX (rule above).

The actor handle returned by SpawnActor is unsafe to store across ticks. The engine may destroy actors via gameplay paths (corpses decay, nests get cleaned up, VFX self-destruct after their animation). Storing a handle and calling K2_DestroyActor on it later will crash the server if the actor was destroyed in the interim (safety rule 9b). For tracking, store a stable identifier (steam, nest code, location) rather than the actor pointer.

## Closing notes

The "what spawns and what doesn't" question has a simple framing: anything where the visual is baked into the BP class default spawns and renders correctly; anything where the visual depends on runtime-attached components (Niagara, dynamic StaticMesh assignment) renders only on listen-servers and standalone (where the rendering pipeline is active), not on the dedicated server.

For the dedicated server use case (where this knowledge matters), commit to BP-asset-based visual content. The `.pak` content path documented in the client-mod-feasibility doc is the route to actually new visuals.
