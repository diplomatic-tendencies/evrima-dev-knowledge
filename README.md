# EVRIMA Dev Knowledge

> English is my third language and the documentation includes AI-assisted translation.

A practical reference for building server-side mods on The Isle EVRIMA. Twenty-plus documents covering UE4SS Lua patterns, crash-class gotchas, mod architectures, C++ side-mod toolchain, asset extraction, and the RCON binary protocol.

Built from real production mod development. Every pattern was confirmed live on a dedicated server. Every gotcha was paid for with at least one crash or a rolled-back mod version. Pull requests and issues welcome.

## Game build 0.21.720 status (the skin-overhaul patch)

The patch broke one core recipe — the customizer write. `SetCustomizerData` (which was always a Server RPC; its declaration did not change) now silently drops server-originated writes: the call succeeds and applies nothing, so skin-writing mods fail **silently**. The replacement recipe, the grown struct with three new paintable regions (teeth, mouth, claws), and the new `PatternIndex` validation trap are in the updated [EVRIMA_Customizer_Field_Map.md](EVRIMA_Customizer_Field_Map.md). The architecture docs that lean on the old write path (SkinMod, CommandBridge's skin verb) carry 0.21.720 notes pointing at the replacement.

The core recipes were re-verified live on 0.21.720 and survived — the state-restore setters, chat, presence — and UE4SS 3.0.1 loads fine; the migration doc lists exactly what was re-tested. Anything not on that list survived loading and smoke testing but has not been individually re-verified, so re-verify before you lean on it. The full triage — including the update-process gotchas that produce fake "server won't boot" panics and the post-patch admin-config gotcha — is in **[EVRIMA_Patch_0.21.720_Migration.md](EVRIMA_Patch_0.21.720_Migration.md)**. Start there if you're coming back after the patch.

## Who this is for

Anyone building UE4SS Lua or C++ side mods for The Isle EVRIMA. Some prior Unreal Engine modding experience is assumed (you know what UFunctions, UClasses, USTRUCTs, and FNames are at a basic level), but no EVRIMA-specific experience is needed; every EVRIMA-specific behavior is documented from scratch.

## Suggested reading order

For a fresh start, read in this order:

1. **[EVRIMA_Lua_Safety_Rules.md](EVRIMA_Lua_Safety_Rules.md)** is the foundation. Fourteen rules covering the crash-causing patterns and the silent-failure traps in EVRIMA Lua modding. Pin this. Everything else assumes you've at least skimmed it.

2. **[EVRIMA_Paths_Reference.md](EVRIMA_Paths_Reference.md)** is the layout reference. Where files live, what each one does, how to launch the server. Quick read.

3. **[EVRIMA_Presence_Registry.md](EVRIMA_Presence_Registry.md)** is the most-reused pattern. Every mod that iterates online players uses this. The engine's own player collections also work for a one-shot read, but the registry's disconnect handling is the proven part, so it stays the production default.

4. **[EVRIMA_State_Restore_Cookbook.md](EVRIMA_State_Restore_Cookbook.md)** is the deepest single document. If you're building anything involving player dino state (save, restore, transform, mutate), this is the recipe.

5. **[EVRIMA_Helpers_Reference.md](EVRIMA_Helpers_Reference.md)** defines every helper function cross-referenced from the other docs (`findGameMode`, `livePawnFromCtrl`, `safeNotify`, the presence registry block, JSON helpers, etc.).

After those five, the rest is reference material. Read as needed.

## Document index

### Foundational

- [EVRIMA_Lua_Safety_Rules.md](EVRIMA_Lua_Safety_Rules.md): fourteen hard rules for what NOT to do in UE4SS Lua on EVRIMA
- [EVRIMA_Paths_Reference.md](EVRIMA_Paths_Reference.md): where everything lives on a server install
- [EVRIMA_Presence_Registry.md](EVRIMA_Presence_Registry.md): production-safe online-player enumeration, plus the engine collections that also work once you read them right
- [EVRIMA_State_Restore_Cookbook.md](EVRIMA_State_Restore_Cookbook.md): full recipe for capturing and restoring dino state
- [EVRIMA_Prime_Elder_Mechanism.md](EVRIMA_Prime_Elder_Mechanism.md): how prime status really works and how to force it from Lua
- [EVRIMA_Species_Swap.md](EVRIMA_Species_Swap.md): live species swap of a player's dino — possession plus the three engine bindings (first working implementation)
- [EVRIMA_Customizer_Field_Map.md](EVRIMA_Customizer_Field_Map.md): the ten color fields, UI label mapping, the 0.21.720 write recipe, and the PatternIndex validation trap
- [EVRIMA_Patch_0.21.720_Migration.md](EVRIMA_Patch_0.21.720_Migration.md): what the skin-overhaul patch broke, what survived, and the update-process gotchas
- [EVRIMA_Chat_System.md](EVRIMA_Chat_System.md): the send and deliver functions, why UpdateChat crashes from Lua, cross-species local delivery
- [EVRIMA_Grouping_Mechanism.md](EVRIMA_Grouping_Mechanism.md): how groups work, where the same-species gate lives, and the server-side recipes for grouping across species
- [EVRIMA_Group_Size.md](EVRIMA_Group_Size.md): the per-species herd/pack cap and its respawn re-stamp
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
- [EVRIMA_SpeciesGroupCap_Architecture.md](EVRIMA_SpeciesGroupCap_Architecture.md): per-species group-size caps with live admin control and respawn-proof enforcement
- [EVRIMA_SpeciesMixpack_Architecture.md](EVRIMA_SpeciesMixpack_Architecture.md): cross-species grouping via the native invite path — the production wrap of the grouping mechanism
- [EVRIMA_KillFeed_Design.md](EVRIMA_KillFeed_Design.md): kill event emitter design plus C++ v2 upgrade path

### Reference catalogs

- [EVRIMA_AI_Spawn_Pairs.md](EVRIMA_AI_Spawn_Pairs.md): 50 verified pawn-plus-AI-controller pairings
- [EVRIMA_Spawnable_Actors.md](EVRIMA_Spawnable_Actors.md): cut species, VFX, fish, plants, gore, nests
- [EVRIMA_StaticMesh_Spawning.md](EVRIMA_StaticMesh_Spawning.md): why runtime mesh assignment fails and what works instead

### Specialized

- [EVRIMA_RCON_Protocol.md](EVRIMA_RCON_Protocol.md): custom binary RCON protocol, 29 confirmed command opcodes
- [EVRIMA_Asset_Extraction.md](EVRIMA_Asset_Extraction.md): CUE4Parse plus Oodle pipeline for offline extraction
- [EVRIMA_Cut_Dino_Enablement.md](EVRIMA_Cut_Dino_Enablement.md): make a cut/withheld playable (Baryonyx, Oviraptor) spawnable server-side, no client mod — client-cooked pak plus runtime class registration
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

**"I want cross-species chat or grouping"**
Chat_System for delivering local lines across species (a C++ job; UpdateChat crashes from Lua). Grouping_Mechanism for forming a group across species, including the native hold-2-to-accept path; it leans on CPP_Side_Mods for the Client RPC and process-event callback. Group_Size for the per-species herd cap. The common thread is that the restriction is client-side and the server functions are species-blind.

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
