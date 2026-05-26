# EVRIMA client-side modding feasibility

This is the honest assessment of what is and isn't feasible for client-side mods on The Isle EVRIMA. Server-side mods (UE4SS Lua, UE4SS C++ side mods) are the main path; this document covers the client side, which is more constrained.

## EAC: yes, it's present

The first thing to verify (and one that's easy to misclaim): Easy Anti-Cheat IS present on EVRIMA. Earlier community discussions occasionally said it wasn't; that was wrong, and tooling has confirmed EAC is loaded on the client side.

EAC's verified-on-EVRIMA behaviors:

- Loaded on the client at game launch (confirmed by the `EasyAntiCheat/` directory and `InstallAntiCheat.bat` shipping with the client install).
- Allows foreign DLLs loaded at the standard proxy slot (`dwmapi.dll`) — verified 2026-05-21 by reaching the main menu with Wine's stub `dwmapi` loaded into that slot. This is how UE4SS would in principle inject client-side on games that allow it.
- Refuses to launch the client when a VM is detected (banner-stage block, verified). Server mode is unaffected.

EAC almost certainly applies its standard suite of additional protections (binary integrity checks, anti-injection checks beyond the proxy slot, etc.); only the behaviors above were directly probed. Don't rely on the absence of any unverified check.

The "allows foreign DLLs at proxy slot" point is what makes client UE4SS theoretically possible. EAC's check is structured to permit the proxy DLL pattern that UE4SS uses for client injection.

## UE4SS on Wine (Linux) does not work

Despite EAC allowing the proxy DLL slot, UE4SS specifically fails on Wine. The failure is an upstream bug in UE4SS's `dwmapi.dll` proxy initialization that produces an access violation in `DllMain` on Wine. This is not an EVRIMA-specific issue; the same bug hits other UE-modded games on Wine.

Until upstream fixes this (the bug has been documented in the UE4SS issue tracker), client-side UE4SS Lua mods are Windows-only.

## VM detection blocks the client

EAC refuses to launch the client in any virtual machine. This is a hard block; there is no workaround that doesn't involve VM-detection evasion (which violates EAC's terms of service and is not a recommendable path).

The practical consequence: testing client-side mods requires bare-metal Windows hardware. A VM dev environment that worked for server-side modding (since the dedicated server has no EAC) does not work for client work.

## What CAN work client-side

The path that works without fighting EAC is `.pak` content mods. These are static asset files (custom textures, custom meshes, custom skins) that the engine loads via its normal asset pipeline. EAC accepts them because the engine's pak-loading is part of the normal game flow, not a separate injection.

The procedure:

1. Build custom assets in Unreal Editor against a UE 5.6 project that mirrors EVRIMA's structure.
2. Package the assets as a `.pak` file (Unreal Editor has a "Cook" or "Package" command that produces this).
3. Place the `.pak` in `<game>/TheIsle/Content/Paks/` on the client.
4. The game loads the new content on launch.

This works for skins, custom meshes (if not gameplay-relevant), retextures, audio packs. It does NOT work for new gameplay mechanics or for anything that would change behavior server-side; the pak is client-side only and the server has no awareness of it.

The cross-platform consequence: `.pak` mods work on any client (Windows, plus future Linux native if EAC ever supports it), do not require UE4SS, and do not require the client to be on a specific OS for development.

## What CANNOT work client-side (without EAC violations)

- Modifying client-side code (e.g. patching the .exe). EAC's standard integrity checks will catch this.
- Injecting custom DLLs outside the proxy slot.
- Running UE4SS Lua client-side on Wine (separate issue — upstream UE4SS bug, not EAC).
- Running the client in a VM.
- Any active EAC bypass / tamper.

Requests for client-side mods like "remove this UI element" or "change this input behavior" or "show this overlay during play" generally hit a "no, EAC doesn't allow it" wall. The exceptions are rare and case-specific.

## The custom content path

For producing custom visual content (custom skins, custom skeleton meshes, custom particle effects) on EVRIMA, the path is:

1. Build the content in Unreal Editor against a UE 5.6 project structured like EVRIMA's.
2. Reference the same materials and textures the base game uses (extract via the procedure in `EVRIMA_Asset_Extraction.md` for visual reference).
3. Package as `.pak` for client distribution.
4. Server-side: spawn instances of the custom BP classes via UE4SS (Lua spawns are fine if the class is BP; C++ side mods if you need custom UCLASSes).

This is the only path to "new visual content that renders both server-spawned and client-visible" on EVRIMA. The server-side-only approach (spawn a vanilla actor with runtime-modified visuals) does not work because the runtime mesh assignment doesn't replicate; see safety rule 9.

A `.pak` plus a small C++ side mod that spawns instances of the new classes is the production pattern for custom visual content. The C++ side mod handles the spawning (and any class-specific logic the new content needs); the .pak handles the visuals.

## Closing notes

EVRIMA's modding surface is asymmetric: the dedicated server is heavily moddable (UE4SS works fully, no EAC, plenty of UFunction surface), but the client is locked down by EAC. The right strategy is to put as much logic server-side as possible (state, persistence, events, scoring), and use `.pak` content for the client-side parts that need to render visual content.

Server-side mods are the low-friction path; that's where most modding effort productively goes. Client `.pak` content is the higher-effort path for new visuals. Client-side scripted features (the equivalent of UE4SS Lua but on the client) are not feasible on EVRIMA in 2026.
