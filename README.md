# EVRIMA Dev Knowledge

> English is my third language and the documentation includes AI-assisted translation.

A practical reference for building server-side mods on The Isle EVRIMA. Twenty-plus documents covering UE4SS Lua patterns, crash-class gotchas, mod architectures, C++ side-mod toolchain, asset extraction, and the RCON binary protocol.

Built from real production mod development. Every pattern was confirmed live on a dedicated server. Every gotcha was paid for with at least one crash or a rolled-back mod version. Pull requests and issues welcome.

## Who this is for

Anyone building UE4SS Lua or C++ side mods for The Isle EVRIMA. Some prior Unreal Engine modding experience is assumed (you know what UFunctions, UClasses, USTRUCTs, and FNames are at a basic level), but no EVRIMA-specific experience is needed; every EVRIMA-specific behavior is documented from scratch.

## Suggested reading order

For a fresh start, read in this order:

1. **[EVRIMA_Lua_Safety_Rules.md](EVRIMA_Lua_Safety_Rules.md)** is the foundation. Twelve rules covering every crash-causing pattern in EVRIMA Lua modding. Pin this. Everything else assumes you've at least skimmed it.

2. **[EVRIMA_Paths_Reference.md](EVRIMA_Paths_Reference.md)** is the layout reference. Where files live, what each one does, how to launch the server. Quick read.

3. **[EVRIMA_Presence_Registry.md](EVRIMA_Presence_Registry.md)** is the most-reused pattern. Every mod that iterates online players uses this. It solves the problem that all the obvious enumeration APIs are broken in different ways.

4. **[EVRIMA_State_Restore_Cookbook.md](EVRIMA_State_Restore_Cookbook.md)** is the deepest single document. If you're building anything involving player dino state (save, restore, transform, mutate), this is the recipe.

5. **[EVRIMA_Helpers_Reference.md](EVRIMA_Helpers_Reference.md)** defines every helper function cross-referenced from the other docs (`findGameMode`, `livePawnFromCtrl`, `safeNotify`, the presence registry block, JSON helpers, etc.).

After those five, the rest is reference material. Read as needed.

## Document index

### Foundational

- [EVRIMA_Lua_Safety_Rules.md](EVRIMA_Lua_Safety_Rules.md): twelve hard rules for what NOT to do in UE4SS Lua on EVRIMA
- [EVRIMA_Paths_Reference.md](EVRIMA_Paths_Reference.md): where everything lives on a server install
- [EVRIMA_Presence_Registry.md](EVRIMA_Presence_Registry.md): safe online-player enumeration (the obvious APIs are broken)
- [EVRIMA_State_Restore_Cookbook.md](EVRIMA_State_Restore_Cookbook.md): full recipe for capturing and restoring dino state
- [EVRIMA_Prime_Elder_Mechanism.md](EVRIMA_Prime_Elder_Mechanism.md): how prime status really works and how to force it from Lua
- [EVRIMA_Customizer_Field_Map.md](EVRIMA_Customizer_Field_Map.md): the seven color fields, UI label mapping, write recipe
- [EVRIMA_EntombBonus_Fix.md](EVRIMA_EntombBonus_Fix.md): elder-stacks counter fix for mutation tier persistence
- [EVRIMA_QuestMutation_Fix.md](EVRIMA_QuestMutation_Fix.md): quest-mutation slot persistence fix
- [EVRIMA_HotReload_Mechanism.md](EVRIMA_HotReload_Mechanism.md): the reload.flag pattern for fast Lua iteration
- [EVRIMA_Helpers_Reference.md](EVRIMA_Helpers_Reference.md): every helper function used across the other docs

### Production mod architectures

- [EVRIMA_DinoStorage_Architecture.md](EVRIMA_DinoStorage_Architecture.md): flagship state-restore mod
- [EVRIMA_BodyDrop_Architecture.md](EVRIMA_BodyDrop_Architecture.md): AI-free corpse spawner
- [EVRIMA_SkinMod_Architecture.md](EVRIMA_SkinMod_Architecture.md): custom skin colors with auto-restore-on-login
- [EVRIMA_PlayerStats_Architecture.md](EVRIMA_PlayerStats_Architecture.md): periodic snapshot emitter
- [EVRIMA_CommandBridge_Architecture.md](EVRIMA_CommandBridge_Architecture.md): NDJSON IPC for bot-to-mod commands
- [EVRIMA_KillFeed_Design.md](EVRIMA_KillFeed_Design.md): kill event emitter design plus C++ v2 upgrade path

### Reference catalogs

- [EVRIMA_AI_Spawn_Pairs.md](EVRIMA_AI_Spawn_Pairs.md): 50 verified pawn-plus-AI-controller pairings
- [EVRIMA_Spawnable_Actors.md](EVRIMA_Spawnable_Actors.md): cut species, VFX, fish, plants, gore, nests
- [EVRIMA_StaticMesh_Spawning.md](EVRIMA_StaticMesh_Spawning.md): why runtime mesh assignment fails and what works instead

### Specialized

- [EVRIMA_RCON_Protocol.md](EVRIMA_RCON_Protocol.md): custom binary RCON protocol, 13 known command codes
- [EVRIMA_Asset_Extraction.md](EVRIMA_Asset_Extraction.md): CUE4Parse plus Oodle pipeline for offline extraction
- [EVRIMA_CPP_Side_Mods.md](EVRIMA_CPP_Side_Mods.md): UE4SS C++ side-mod toolchain
- [EVRIMA_Client_Mod_Feasibility.md](EVRIMA_Client_Mod_Feasibility.md): what works and doesn't on the client side (EAC constraints)

## Reading patterns by goal

**"I want to build a custom admin command for the server"**
Lua_Safety_Rules, Paths_Reference, Presence_Registry, Helpers_Reference. That's 90% of what you need. Add HotReload_Mechanism if you want fast dev iteration.

**"I want to build a save/restore feature for dinos"**
All five foundational docs, then DinoStorage_Architecture. State_Restore_Cookbook is the main reference; EntombBonus and QuestMutation fix docs are the deep dives on the two subtleties that bit me twice.

**"I want to spawn things in the world"**
Lua_Safety_Rules (specifically rules 9 and 9b), Spawnable_Actors, AI_Spawn_Pairs, BodyDrop_Architecture, StaticMesh_Spawning. The combination tells you what spawns, what doesn't, and why.

**"I want to integrate with a Discord bot"**
PlayerStats_Architecture, CommandBridge_Architecture, KillFeed_Design. The NDJSON event-stream pattern plus the bidirectional IPC layer cover most bot integration needs.

**"I want to do custom visual content (skins, particles, meshes)"**
Asset_Extraction, Spawnable_Actors, Client_Mod_Feasibility. The path is `.pak` content built in Unreal Editor plus a server-side spawner; documented but high-cost work.

**"I want to use C++ for capabilities Lua can't reach"**
CPP_Side_Mods. Roughly a 6-hour first session to bring the toolchain up. Subsequent mods are half a day each plus the feature work.

## Contributing

Corrections and new findings welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for the contribution guide. The short version: open an issue for "this is wrong" or "I observed X," open a PR for additions. Verify behavior claims on a live server before submitting where practical.

The docs are versioned by the natural git history. Behavioral claims may need re-verification across major EVRIMA updates; if you find a claim that no longer holds, that's exactly the kind of correction an issue should call out.

## License

[CC-BY 4.0](LICENSE). Free to copy, modify, and redistribute including for commercial use, with attribution.

Code snippets (Lua, C++, PowerShell, JSON schemas) are additionally available under MIT for use in commercial or closed-source projects. See the [LICENSE](LICENSE) file for the dual-licensing terms.

## Version note

The patterns here reflect EVRIMA dedicated server modding as verified against UE 5.6 and UE4SS v3.0.1, mid-2026. Some specific behaviors (the SetAdminCred heartbeat cadence, the SetSlot batching limit, the specific UFunctions that fire vs don't) may change with EVRIMA updates; re-verify before relying on edge-case behavior.
