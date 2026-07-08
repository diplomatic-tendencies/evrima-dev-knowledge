# Spawning static meshes server-side on EVRIMA

Short practical guide for spawning visible static meshes (decor, nests, gore, plants, skeletons) from UE4SS Lua on The Isle EVRIMA dedicated server.

---

## The one thing you need to know

**Don't try to use `AStaticMeshActor` + `SetStaticMesh()` from Lua.** It looks like it should work, but the static mesh assignment is NOT a replicated UProperty in Unreal, the server has your cube, the client sees an invisible empty actor. The `SetReplicates(true)` flag doesn't help because there's no replication path for the mesh field.

**DO this instead**: spawn a Blueprint class that has the mesh baked into its class default. "Spawn class X" replicates fine because the client already knows what mesh class X uses by name.

This is what BodyDrop does for corpses, and what every working "spawn a thing" mod on EVRIMA does.

---

## The basic spawn recipe

```lua
-- 1. Resolve the class by full object path
local classObj = StaticFindObject("/Game/TheIsle/.../BP_SomeThing.BP_SomeThing_C")
if classObj == nil then return false, "class not loaded" end

-- 2. Get a UWorld reference (any pawn or game-mode works)
local world = pawn:GetWorld()  -- or gameMode:GetWorld()
if world == nil then return false, "no world" end

-- 3. Spawn it
local loc = { X = 0, Y = 0, Z = 0 }
local rot = { Pitch = 0, Yaw = 0, Roll = 0 }
local okSpawn, actor = pcall(function() return world:SpawnActor(classObj, loc, rot) end)
if not okSpawn or actor == nil then return false, "spawn threw" end

-- 4. CRITICAL: validate the actor wrapper is not a nullptr
-- Terrain collision causes SpawnActor to return a non-nil Lua wrapper
-- around a NULLPTR UObject. Calling methods on that crashes the server.
local addr
pcall(function() addr = actor:GetAddress() end)
if addr == nil or addr == 0 then
    return false, "SpawnActor returned nullptr (terrain or invalid location)"
end

-- 5. Flip network flags so clients receive the actor.
-- SetReplicates + ForceNetUpdate are what a normal actor needs; it then
-- replicates on native distance relevancy. Do NOT reflexively add
-- bAlwaysRelevant (see Gotcha 3) — for a high-count/replicated spawn it is a
-- client load-in-crash amplifier, and it does nothing for ordinary decor.
pcall(function() actor:SetReplicates(true) end)
pcall(function() actor:ForceNetUpdate() end)

return true, actor
```

If `SpawnActor` fails (nullptr), resample a fresh XY near the target and re-derive ground (see Gotcha 2). Do not retry with an untraced vertical offset — an actor spawned into the void below the landscape reports success too.

---

## Useful BP classes you can spawn

All paths verified to exist on the EVRIMA build. Get the full path by appending `.BP_<Name>_C` to the file path.

### Nests

`/Game/TheIsle/Core/Nests/BP_Nest_*`, 11 variants:
- `BP_Nest_Mound_Large_H` (large herbivore nest)
- `BP_Nest_Mound_Medium_H`, `BP_Nest_Mound_Small_H`
- `BP_Nest_Burrow_*` (burrow variants)
- Plus carnivore variants

These have functional gameplay (eggs, breeding mechanics). Use cautiously.

### Gore decor (`/Game/TheIsle/Core/PickablePieces/`)

Physics-enabled chunks for decoration or gore scenes:
- `BP_GoreEggCarnivoreSmall`, `BP_GoreEggCarnivoreMedium`, `BP_GoreEggCarnivoreLarge`
- `BP_GoreEggHerbivoreSmall`, `BP_GoreEggHerbivoreMedium`, `BP_GoreEggHerbivoreLarge`
- `BP_GoreHeart`, `BP_GoreIntestines`
- `BP_GoreLungeLeft`, `BP_GoreLungeRight`
- `BP_GoreMeat`, `BP_GoreMeatStretch`, `BP_GoreStomach`

Edible by carnivores. Physics-active so they fall and rest.

### Edible plants (`/Game/TheIsle/Core/Spawnables/EdiblePlants/`)

30+ variants, herbivore food + decor:
- Common: `BP_Agave_*`, `BP_Fiddlehead_*`, `BP_Fireweed_*`, `BP_Marigold_*`, `BP_Russula_*`, `BP_Sumac_*`, `BP_Trillium_*`
- Tropical trees (StaticSpawner pattern, these spawn a base + interactive pieces):
  - `BP_BananaTreeStaticSpawner`
  - `BP_BrazilnutsTreeStaticSpawner`
  - `BP_CashewStaticSpawner`
  - `BP_CoconutTreeStaticSpawner`
  - `BP_JackfruitTreeStaticSpawner`
  - `BP_MangoTreeStaticSpawner`
  - `BP_PapayaTreeStaticSpawner`
  - `BP_PumpkinStaticSpawner`
  - `BP_HornedMelonStaticSpawner`
  - `BP_VariegatedOrangeStaticSpawner`
- Mushrooms: `BP_ChanterelleMushroom_*`, `BP_JuvMushroom_*`
- Apollan trifoliums (lore plants): `BP_AzureApollanK*`, `BP_CrimsonApollanK*`, `BP_VioletApollanK*`
- Roots: `BP_RadishRoot_*`, `BP_WildPotatoRoot_*`, `BP_SunchokeRoot_*`

### Dino corpse decor

Spawning a dino BP and immediately killing/ragdolling it produces a usable corpse mesh (see BodyDrop pattern for the full recipe). Cheaper than running live AI. The species set BodyDrop ships with:

`/Game/TheIsle/Core/Characters/Dinosaurs/<Species>/BP_<Species>.BP_<Species>_C`

Species: allosaurus, beipiaosaurus, carnotaurus, ceratosaurus, compsognathus, deinosuchus, diabloceratops, dilophosaurus, dryosaurus, gallimimus, herrerasaurus, hypsilophodon, maiasaura, omniraptor, pachycephalosaurus, pteranodon, stegosaurus, tenontosaurus, triceratops, troodon, tyrannosaurus.

### Skeleton meshes (untested but probable)

Every dino has a `SK_<Species>_Bones` skeletal mesh asset at `/Game/TheIsle/Characters/Dinosaurs/<Species>/Mesh/`. These are the skeleton-only mesh assets used by death/decay states.

You can't spawn them directly as an actor (they're meshes, not BPs), you'd need a custom BP wrapper that uses one of these meshes as its baked class default. Building that BP requires Unreal Editor and a `.pak` asset pack.

### Burrow construction kit (`/Game/TheIsle/BurrowConcept/`)

Unreleased prototype set, could be assembled into a custom underground sandbox:
- `BP_BurrowActor` (container)
- `BP_BurrowEntrance` (surface entry)
- `BP_BurrowConnectorCorridor` (hallway)
- `BP_BurrowConnectorNoExit` (dead end)
- `BP_BurrowConnectorRoomAllSides` (4-way junction)
- `BP_GeneratedVoxelMeshActor` + `BPI_VoxelInterface` (procedural voxel terrain)
- `BP_InternalInteractor` (in-burrow interactables)

### Misc

- `BP_NameWidgetActor`, floating name tags
- `BP_AmbZone`, ambient sound zone
- `BP_SpawnBounds`, defines spawn zones
- `BP_AdminPawn`, admin spectator pawn
- `BP_FishSchool_*`, fish school spawners (6 variants)

---

## Gotchas

1. **Always validate `actor:GetAddress() ~= 0`** after `SpawnActor`. Terrain collision returns a non-nil Lua wrapper around a nullptr UObject, and calling methods on it crashes the server with a 30-second-delayed AV that `pcall` cannot catch.

2. **Find real ground; don't spawn at a blind vertical offset.** EVRIMA's heightmap is uneven, and spawning at a naive ground Z can clip the actor into a hill — but the fix is a downward trace to the true surface at your XY, not a fixed `+1500`. A blind offset is the trap the corpse-spawner (BodyDrop) had to remove: `SpawnActor` succeeds in the empty void *below* the landscape, so an untraced offset over the wrong spot buries the actor under the map. Trace from a high ceiling Z straight down, take the hit, spawn just above it. Note the old "+1500 and let it fall" advice only ever did anything for a **physics/ragdoll** actor (a corpse); a static decor mesh spawned at +1500 just floats at +1500.

3. **Network flags for client visibility:**
   ```lua
   actor:SetReplicates(true)
   actor:ForceNetUpdate()
   ```
   These two are what an actor needs; it then replicates on native distance relevancy like anything else. `bAlwaysRelevant = true` is **optional and usually wrong** here: it forces the actor into every joining client's initial replication burst, which at scale (many spawned actors, e.g. corpses) is a documented client load-in crash — BodyDrop v004 removed it for exactly that reason. Reach for it only for a single, must-always-see actor, never as a blanket "make it visible" flag.

4. **DO NOT call `K2_DestroyActor`** to clean up server-side. If gameplay destroyed the actor independently (corpse-eaten, decay timer, etc.), your destroy call fires a native AV on freed memory. Even checking `GetAddress() ~= 0` first isn't safe, freed-but-not-yet-reused memory passes the check and the destroy itself crashes. For cleanup, restart the server. For tracking, use a write-once table you read for stats but never use to destroy from.

5. **Don't call `FindFirstOf("BP_Class_Name")` on a class with no live instances.** That call can crash UE4SS internally. Use `StaticFindObject("/Game/full/path.BP_Name_C")` instead, that always works as long as the class is loaded into UObject space.

6. **Custom meshes need a `.pak`, not Lua.** If you want to add a NEW mesh that isn't in the game already, you have to build a custom BP in Unreal Editor wrapping that mesh, package it as a `.pak` file, and drop it in the server's `Paks/` directory. Server-side Lua can only spawn classes that already exist in the game build.

---

## What about VFX / particle effects?

Mostly doesn't work server-side from Lua. The dedicated server has no renderer, so `UNiagaraFunctionLibrary:SpawnSystemAtLocation` returns nullptr.

BP-wrapped effect actors (like `BP_SmiteEffect`, `BP_BloodSplat`, `BP_Footprint`) DO spawn via `world:SpawnActor`, but their effect playback is gated by client-side execution paths the server can't trigger. Result: invisible empty actors.

The ONE exception is `BP_HallucinationFog`, it's wired into the venom/dilo hallucination gameplay state machine, so its replication path is fully built. You can spawn that and clients will see the fog. Other 7+ effect actors spawn-but-don't-render.

For real visual effects mods, you need a `.pak`-based asset pack with custom Niagara systems baked in.
