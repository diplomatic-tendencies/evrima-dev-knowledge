# EVRIMA AI Spawn Pair Catalog

Every pawn species + AI controller pair that produces a working AI brain when spawned server-side.

**Verification**: live-tested 2026-05-22 on The Isle EVRIMA dedicated server.
"Works" = pawn spawned, controller possessed, AI brain actually pathfinds the pawn (movement delta > 500 units over a 10-second observation window).

**Total verified**: 50 working pairs across 25 dinosaur species + 7 animal species.

## How to spawn AI server-side (UE4SS Lua)

```lua
local pawnCls = StaticFindObject("/Game/.../BP_Tyrannosaurus.BP_Tyrannosaurus_C")
local ctrlCls = StaticFindObject("/Script/TheIsle.TIAIRexController")
local world = gameMode:GetWorld()

-- Try a few offsets if first attempt hits terrain
local pawn = world:SpawnActor(pawnCls, {X=loc.X, Y=loc.Y, Z=loc.Z+500}, {Pitch=0, Yaw=0, Roll=0})
-- IMPORTANT: validate pawn:GetAddress() ~= 0 (terrain collision returns nullptr wrapper)

local ctrl = world:SpawnActor(ctrlCls, {X=loc.X, Y=loc.Y, Z=loc.Z}, {Pitch=0, Yaw=0, Roll=0})
ctrl:Possess(pawn)
-- AI brain boots automatically on Possess; no explicit start needed
```

Default spawn growth is hatchling (~0.05). For adult AI:
```lua
pawn:SetGrowth(1.0)
pawn:SetHealth(pawn:GetMaxHealth())
pawn:SetFood(pawn:GetMaxFood())
pawn:SetThirst(pawn:GetMaxThirst())
-- Note: SetGrowth wipes vitals to max, so call vital setters AFTER SetGrowth
```

---

## Dinosaurs, native controllers (no BP wrapper needed)

These use direct `/Script/TheIsle.<class>` paths. The C++ class is the controller itself.

| Species | Pawn class | Controller class |
|---|---|---|
| Tyrannosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Tyrannosaurus/BP_Tyrannosaurus.BP_Tyrannosaurus_C` | `/Script/TheIsle.TIAIRexController` |
| Dilophosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Dilophosaurus/BP_Dilophosaurus.BP_Dilophosaurus_C` | `/Script/TheIsle.TIAIDilophosaurusController` |
| Carnotaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Carnotaurus/BP_Carnotaurus.BP_Carnotaurus_C` | `/Script/TheIsle.TIAICarnotaurus` |
| Ceratosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Ceratosaurus/BP_Ceratosaurus.BP_Ceratosaurus_C` | `/Script/TheIsle.TIAICeratosaurusController` |
| Tenontosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Tenontosaurus/BP_Tenontosaurus.BP_Tenontosaurus_C` | `/Script/TheIsle.TIAITenontosaurusController` |
| Diabloceratops | `/Game/TheIsle/Core/Characters/Dinosaurs/Diabloceratops/BP_Diabloceratops.BP_Diabloceratops_C` | `/Script/TheIsle.TIAIDiabloceratopsController` |
| Dryosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Dryosaurus/BP_Dryosaurus.BP_Dryosaurus_C` | `/Script/TheIsle.TIAIDryosaurusController` |
| Gallimimus | `/Game/TheIsle/Core/Characters/Dinosaurs/Gallimimus/BP_Gallimimus.BP_Gallimimus_C` | `/Script/TheIsle.TIAIGallimimusController` |
| Hypsilophodon | `/Game/TheIsle/Core/Characters/Dinosaurs/Hypsilophodon/BP_Hypsilophodon.BP_Hypsilophodon_C` | `/Script/TheIsle.TIAIHypsilophodon` |
| Omniraptor | `/Game/TheIsle/Core/Characters/Dinosaurs/Omniraptor/BP_Omniraptor.BP_Omniraptor_C` | `/Script/TheIsle.TIAIOmniraptor` |
| Psittacosaurus (Coastal) | `/Game/TheIsle/Core/Characters/Dinosaurs/Psittacosaurus/BP_Psittacosaurus_Coastal.BP_Psittacosaurus_Coastal_C` | `/Script/TheIsle.TIAIPsittacosaurusController` |
| Pteranodon | `/Game/TheIsle/Core/Characters/Dinosaurs/Pteranodon/BP_Pteranodon.BP_Pteranodon_C` | `/Script/TheIsle.TIAIPteranodon` |
| Deinosuchus | `/Game/TheIsle/Core/Characters/Dinosaurs/Deinosuchus/BP_Deinosuchus.BP_Deinosuchus_C` | `/Script/TheIsle.TIAIDeinosuchus` |

Note: `TIAICarnotaurus`, `TIAIHypsilophodon`, `TIAIOmniraptor`, `TIAIPteranodon`, `TIAIDeinosuchus` are mis-named in the game's class hierarchy (no "Controller" suffix), but they ARE AI controllers per their parent class chain.

## Dinosaurs, BP controllers

These use the Blueprint-wrapped controllers. Equivalent to native versions where both exist; required for species with no native controller (Compsognathus, Pterodactylus).

| Species | Pawn class | Controller class |
|---|---|---|
| Compsognathus | `/Game/TheIsle/Core/Characters/Dinosaurs/Compsognathus/BP_Compsognathus.BP_Compsognathus_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Compsognathus_Controller.BP_AI_Compsognathus_Controller_C` |
| Gallimimus | `/Game/TheIsle/Core/Characters/Dinosaurs/Gallimimus/BP_Gallimimus.BP_Gallimimus_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Galli_Controller.BP_AI_Galli_Controller_C` |
| Pterodactylus | `/Game/TheIsle/Core/Characters/Dinosaurs/Pterodactylus/BP_Pterodactylus.BP_Pterodactylus_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Pterodactylus_Controller.BP_AI_Pterodactylus_Controller_C` |
| Pteranodon | `/Game/TheIsle/Core/Characters/Dinosaurs/Pteranodon/BP_Pteranodon.BP_Pteranodon_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Pteranodon_Controller.BP_AI_Pteranodon_Controller_C` |
| Deinosuchus | `/Game/TheIsle/Core/Characters/Dinosaurs/Deinosuchus/BP_Deinosuchus.BP_Deinosuchus_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Deinosuchus_Controller.BP_AI_Deinosuchus_Controller_C` |
| Diabloceratops | `/Game/TheIsle/Core/Characters/Dinosaurs/Diabloceratops/BP_Diabloceratops.BP_Diabloceratops_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Diabloceratops_Controller.BP_AI_Diabloceratops_Controller_C` |
| Dryosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Dryosaurus/BP_Dryosaurus.BP_Dryosaurus_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Dino_Dryosaurus_Controller.BP_AI_Dino_Dryosaurus_Controller_C` |
| Ceratosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Ceratosaurus/BP_Ceratosaurus.BP_Ceratosaurus_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Cerato_Controller.BP_AI_Cerato_Controller_C` |
| Hypsilophodon | `/Game/TheIsle/Core/Characters/Dinosaurs/Hypsilophodon/BP_Hypsilophodon.BP_Hypsilophodon_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Hypsilophodon_Controller.BP_AI_Hypsilophodon_Controller_C` |
| Omniraptor | `/Game/TheIsle/Core/Characters/Dinosaurs/Omniraptor/BP_Omniraptor.BP_Omniraptor_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Omniraptor_Controller.BP_AI_Omniraptor_Controller_C` |
| Psittacosaurus (Coastal) | `/Game/TheIsle/Core/Characters/Dinosaurs/Psittacosaurus/BP_Psittacosaurus_Coastal.BP_Psittacosaurus_Coastal_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Psittacosaurus_Coastal_Controller.BP_AI_Psittacosaurus_Coastal_Controller_C` |
| Psittacosaurus (Highlands) | `/Game/TheIsle/Core/Characters/Dinosaurs/Psittacosaurus/BP_Psittacosaurus_Highlands.BP_Psittacosaurus_Highlands_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Psittacosaurus_Controller.BP_AI_Psittacosaurus_Controller_C` |
| Psittacosaurus (Plains) | `/Game/TheIsle/Core/Characters/Dinosaurs/Psittacosaurus/BP_Psittacosaurus_Plains.BP_Psittacosaurus_Plains_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Psittacosaurus_Controller.BP_AI_Psittacosaurus_Controller_C` |

## Cross-species pairs, for species with no dedicated AI controller

| Species | Pawn class | Borrowed controller |
|---|---|---|
| Triceratops | `/Game/TheIsle/Core/Characters/Dinosaurs/Triceratops/BP_Triceratops.BP_Triceratops_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Diabloceratops_Controller.BP_AI_Diabloceratops_Controller_C` |
| Stegosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Stegosaurus/BP_Stegosaurus.BP_Stegosaurus_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Diabloceratops_Controller.BP_AI_Diabloceratops_Controller_C` |
| Troodon | `/Game/TheIsle/Core/Characters/Dinosaurs/Troodon/BP_Troodon.BP_Troodon_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Compsognathus_Controller.BP_AI_Compsognathus_Controller_C` |
| Allosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Allosaurus/BP_Allosaurus.BP_Allosaurus_C` | `/Script/TheIsle.TIAIRexController` |
| Herrerasaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Herrerasaurus/BP_Herrerasaurus.BP_Herrerasaurus_C` | `/Script/TheIsle.TIAIOmniraptor` |
| Beipiaosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Beipiaosaurus/BP_Beipiaosaurus.BP_Beipiaosaurus_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Compsognathus_Controller.BP_AI_Compsognathus_Controller_C` |
| Maiasaura | `/Game/TheIsle/Core/Characters/Dinosaurs/Maiasaura/BP_Maiasaura.BP_Maiasaura_C` | `/Script/TheIsle.TIAITenontosaurusController` |
| Pachycephalosaurus | `/Game/TheIsle/Core/Characters/Dinosaurs/Pachycephalosaurus/BP_Pachycephalosaurus.BP_Pachycephalosaurus_C` | `/Game/TheIsle/Core/AI/Controllers/Dinos/BP_AI_Diabloceratops_Controller.BP_AI_Diabloceratops_Controller_C` |

## Animals

### Native controllers

| Species | Pawn class | Controller class |
|---|---|---|
| Boar | `/Game/TheIsle/Core/Characters/Animals/Boar/BP_Boar.BP_Boar_C` | `/Script/TheIsle.TIAIBoarController` |
| Chicken | `/Game/TheIsle/Core/Characters/Animals/Chicken/BP_Chicken.BP_Chicken_C` | `/Script/TheIsle.TIAIChickenController` |
| Crab | `/Game/TheIsle/Core/Characters/Animals/Crab/BP_Crab.BP_Crab_C` | `/Script/TheIsle.TIAICrabController` |
| Goat | `/Game/TheIsle/Core/Characters/Animals/Goat/BP_goat.BP_Goat_C` | `/Script/TheIsle.TIAIGoatController` |
| Rabbit | `/Game/TheIsle/Core/Characters/Animals/Rabbit/BP_Rabbit.BP_Rabbit_C` | `/Script/TheIsle.TIAIRabbitController` |
| Seaturtle | `/Game/TheIsle/Core/Characters/Animals/Seaturtle/BP_Seaturtle.BP_Seaturtle_C` | `/Script/TheIsle.TIAISeaturtleController` |
| Bullfrog | `/Game/TheIsle/Core/Characters/Animals/Bullfrog/BP_Bullfrog.BP_Bullfrog_C` | `/Script/TheIsle.TIAIFrogController` |

### BP controllers

| Species | Pawn class | Controller class |
|---|---|---|
| Boar | `/Game/TheIsle/Core/Characters/Animals/Boar/BP_Boar.BP_Boar_C` | `/Game/TheIsle/Core/AI/Controllers/Animals/BP_AI_Boar_Controller.BP_AI_Boar_Controller_C` |
| Chicken | `/Game/TheIsle/Core/Characters/Animals/Chicken/BP_Chicken.BP_Chicken_C` | `/Game/TheIsle/Core/AI/Controllers/Animals/BP_AI_Chicken_Controller.BP_AI_Chicken_Controller_C` |
| Crab | `/Game/TheIsle/Core/Characters/Animals/Crab/BP_Crab.BP_Crab_C` | `/Game/TheIsle/Core/AI/Controllers/Animals/BP_AI_Crab_Controller.BP_AI_Crab_Controller_C` |
| Deer | `/Game/TheIsle/Core/Characters/Animals/Deer/BP_Deer.BP_Deer_C` | `/Game/TheIsle/Core/AI/Controllers/Animals/BP_AI_Deer_Controller.BP_AI_Deer_Controller_C` |
| Deer (Baby) | `/Game/TheIsle/Core/Characters/Animals/Deer/BP_Deer_Baby.BP_Deer_Baby_C` | `/Game/TheIsle/Core/AI/Controllers/Animals/BP_AI_Deer_Baby_Controller.BP_AI_Deer_Baby_Controller_C` |
| Goat | `/Game/TheIsle/Core/Characters/Animals/Goat/BP_goat.BP_Goat_C` | `/Game/TheIsle/Core/AI/Controllers/Animals/BP_AI_Goat_Controller.BP_AI_Goat_Controller_C` |
| Rabbit | `/Game/TheIsle/Core/Characters/Animals/Rabbit/BP_Rabbit.BP_Rabbit_C` | `/Game/TheIsle/Core/AI/Controllers/Animals/BP_AI_Rabbit_Controller.BP_AI_Rabbit_Controller_C` |
| Seaturtle | `/Game/TheIsle/Core/Characters/Animals/Seaturtle/BP_Seaturtle.BP_Seaturtle_C` | `/Game/TheIsle/Core/AI/Controllers/Animals/BP_AI_SeaTurtle_Controller.BP_AI_SeaTurtle_Controller_C` |
| Bullfrog | `/Game/TheIsle/Core/Characters/Animals/Bullfrog/BP_Bullfrog.BP_Bullfrog_C` | `/Game/TheIsle/Core/AI/Controllers/Animals/BP_AI_Animal_Bullfrog_Controller.BP_AI_Animal_Bullfrog_Controller_C` |

---

## Not working / not tested

| Species | Reason |
|---|---|
| Elite Catfish (`BP_Elite_Fish_CatFish_C`) | Possessed but didn't move. Fish AI needs the pawn to be in water at spawn. |
| Elite Coelacanth (`BP_Elite_Fish_Coelacanth_C`) | Same as above. |
| School fish (5 variants: Catfish, Hoplosternum, Longear, MuskelLunge, RainbowFish) | same water-context requirement. |
| Kentrosaurus | Class loads but `SpawnActor` returns nullptr - confirmed stub class, no functional default subobjects. |
| Parasaurolophus | Same as Kentrosaurus - stub class. |

---

## Native AI controllers without a known pawn

These C++ AI controllers exist but the corresponding pawn class wasn't located:

- `/Script/TheIsle.TIAILizardController` (no `BP_Lizard` found)
- `/Script/TheIsle.TIAIDragonflyController` (no `BP_Dragonfly` found)
- `/Script/TheIsle.TIAIGrasshopperController` (no `BP_Grasshopper` found)
- `/Script/TheIsle.TIAIClamController` (no `BP_Clam` found)
- `/Script/TheIsle.TIAIScavengerController` (base class, scavenger behavior, needs pawn)
- `/Script/TheIsle.TIAIFlyingScavengerController` (vulture-style aerial scavenger, no `BP_Vulture` found)

These are probably either cut content with unfinished pawn assets, or the pawns exist under an unusual naming convention that wasn't discovered. Future investigation could try `StaticFindObject("/Script/TheIsle.TIAILizard")` etc. for native-only pawn classes.

---

## Gotchas (important for anyone implementing this)

1. **Cache pawn wrappers carefully**. If gameplay destroys the pawn (player kill, corpse cleanup, etc.) and you later call methods on the cached wrapper, you get a delayed native AV that crashes the server. Always validate `pawn:GetAddress() ~= 0` before any method call, and even then wrap each call in `pcall`.

2. **`controller.BrainComponent` reads as nullptr** via UE4SS dot-access, even when the AI is verifiably working. Don't gate on this field as a "did it work" check. Use a movement check (location delta over time) instead.

3. **`bStartAILogicOnPossess` reads as false for almost everything** but AI still boots. Unreliable signal.

4. **Default spawn growth is hatchling**. Call `SetGrowth(1.0)` AFTER `Possess()` for adult AI. Then re-apply vitals (`SetHealth/SetFood/SetThirst/SetStamina`) because `SetGrowth` resets them to max-of-the-new-size.

5. **Server-spawned actors need network flags** for clients to see them:
   ```lua
   pawn:SetReplicates(true)
   pawn.bAlwaysRelevant = true
   pawn:ForceNetUpdate()
   ```

6. **DON'T try to clean up AI from Lua** via `K2_DestroyActor`. If gameplay already destroyed the pawn, your destroy call crashes the server. For cleanup, restart the server. Track spawned actors for read-only purposes only; never for destroy paths.
