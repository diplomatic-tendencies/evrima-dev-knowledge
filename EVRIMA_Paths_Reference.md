# EVRIMA dedicated server paths

Quick reference for where everything lives on a standard Windows dedicated server install. Adjust the root path (`<game>`) to wherever your server is installed; everything below the root is consistent.

## Server install layout

```
<game>/
├── TheIsleServer.exe              # The launcher / wrapper exe
├── TheIsle/
│   ├── Binaries/
│   │   └── Win64/
│   │       ├── TheIsleServer-Win64-Shipping.exe   # The actual server binary
│   │       ├── UE4SS.dll                          # UE4SS runtime (if installed)
│   │       ├── UE4SS-settings.ini                 # UE4SS config
│   │       ├── UE4SS.log                          # UE4SS log (recreated each boot)
│   │       ├── dwmapi.dll                         # UE4SS's proxy DLL
│   │       └── Mods/                              # All UE4SS mods live here
│   │           ├── mods.txt                       # Mod enable/disable list
│   │           ├── enabled.txt                    # Alternative enable list
│   │           └── <ModName>/                     # One per mod
│   │               ├── Scripts/
│   │               │   └── main.lua               # Lua mod entry
│   │               ├── dlls/
│   │               │   └── main.dll               # C++ side mod (if any)
│   │               └── Saved/                     # Per-mod runtime storage
│   ├── Content/
│   │   └── Paks/                                  # Game .pak content files
│   ├── Config/
│   │   ├── DefaultGame.ini                        # Default game config
│   │   └── DefaultEngine.ini                      # Default engine config
│   └── Saved/
│       ├── Config/
│       │   └── WindowsServer/
│       │       ├── Game.ini                       # Server's runtime Game.ini overrides
│       │       └── Engine.ini                     # Server's runtime Engine.ini
│       ├── Logs/                                  # Server log output
│       │   └── TheIsle.log                        # Main server log (rotated)
│       └── PlayerData/                            # Per-player .sav files
│           └── <steam>.sav                        # Engine-managed; do not edit
└── Engine/                                        # Stock UE engine assets
```

## Critical paths

| Purpose | Path |
|---|---|
| Server binary (where you launch from) | `<game>/TheIsleServer.exe` |
| Actual shipping binary | `<game>/TheIsle/Binaries/Win64/TheIsleServer-Win64-Shipping.exe` |
| UE4SS runtime DLL | `<game>/TheIsle/Binaries/Win64/UE4SS.dll` |
| UE4SS log | `<game>/TheIsle/Binaries/Win64/UE4SS.log` |
| Mods directory | `<game>/TheIsle/Binaries/Win64/Mods/` |
| Mod enable list | `<game>/TheIsle/Binaries/Win64/Mods/mods.txt` |
| Server config | `<game>/TheIsle/Saved/Config/WindowsServer/Game.ini` |
| Server log | `<game>/TheIsle/Saved/Logs/TheIsle.log` |
| Per-player saves | `<game>/TheIsle/Saved/PlayerData/<steam>.sav` |
| Pak content | `<game>/TheIsle/Content/Paks/` |

## Process identification

EVRIMA spawns TWO processes when the server starts:

1. `TheIsleServer.exe` (the launcher/wrapper).
2. `TheIsleServer-Win64-Shipping.exe` (the actual shipping binary that does all the work).

When killing the server for redeploy of a UE4SS mod DLL, you need to kill BOTH. Using PowerShell:

```powershell
Get-Process | Where-Object { $_.Name -match "^TheIsleServer" } | Stop-Process -Force
```

A `Stop-Process -Name "TheIsleServer"` matches only the wrapper; the shipping binary keeps running and holds DLL file locks open. This caught me out during C++ side mod redeploys.

Wait at least 5 to 10 seconds after killing before copying new DLLs into the Mods folder. Windows takes a beat to release file handles.

## UE4SS mod layout

Each mod gets its own directory under `Mods/`:

```
Mods/MyMod/
├── Scripts/
│   └── main.lua                # Lua mod entry, loaded if mod is enabled
├── dlls/
│   └── main.dll                # Optional C++ side mod DLL
└── Saved/                      # Mod's per-run storage; create as needed
    └── (whatever your mod writes)
```

The `Scripts/main.lua` is the Lua entry point. UE4SS calls it on boot for any mod listed as enabled in `mods.txt` (or `enabled.txt`).

The `dlls/main.dll` is the C++ side mod entry. The DLL MUST be named `main.dll`. It MUST export `start_mod` and `uninstall_mod` as `extern "C"`. See `EVRIMA_CPP_Side_Mods.md` for the build process.

A single mod can have both a Lua component (in `Scripts/`) and a C++ component (in `dlls/`). They share the same `Saved/` directory.

## mods.txt format

One line per mod. The format is `ModName : Enabled` where Enabled is 0 or 1:

```
DinoStorage : 1
BodyDrop : 1
SkinMod : 1
SomeDeprecatedProbeMod : 0
```

Comments start with `;` and apply to the whole line. Blank lines are fine. The order in mods.txt is the load order; mods load top to bottom.

UE4SS reads both `mods.txt` and `enabled.txt`. If both exist, both are processed. The convention is to use `mods.txt` for the curated list and let `enabled.txt` be auto-generated by UE4SS if needed.

## UE4SS log format

Lines look like:

```
[2026-05-23 19:35:21.0791728] Starting C++ mod 'KillFeedMod'
[2026-05-23 19:35:21.0813019] [KillFeedMod] constructor (v0.1.0-hello-world)
[2026-05-23 19:35:24.3016998] [Lua] [DinoStorage] Boot; version=v021-mut-persistence-complete
```

The log is created fresh on every server boot. Mods' `print()` and C++ `Output::send` calls all go here.

For grep-friendly searches, prefix every log line in your mod with `[YourModName]`. That's how every example in these docs uses the convention.

## Server startup invocation

A typical server start command:

```powershell
& "<game>/TheIsleServer.exe" `
    -log `
    -ini:Engine:[EpicOnlineServices]:DedicatedServerClientId=<YOUR_CLIENT_ID> `
    -ini:Engine:[EpicOnlineServices]:DedicatedServerClientSecret=<YOUR_CLIENT_SECRET> `
    -Port=7778
```

The `-log` flag enables console logging output.

The Epic OnlineServices credentials are per-server; obtain them from the game's official server-hosting documentation. They authenticate the server to Epic's online services.

The port is the game traffic port. Default 7777; many servers use 7778 to avoid conflicts. The RCON port is separate (default 8888 in most server configs).

## Game.ini relevant keys

The `Saved/Config/WindowsServer/Game.ini` overrides the defaults shipped with the game. Common keys:

```ini
[/Script/TheIsle.TIGameStateBase]
AdminsSteamIDs=76561198XXXXXXX,76561198YYYYYYY

[/Script/TheIsle.TIGameModeBase]
GameRules=Survival
ServerPassword=secret  # if you want a password gate
```

The `AdminsSteamIDs` list is what the `SetAdminCred` heartbeat's bool parameter reflects. Adding a steam to this list and rebooting causes that player's `SetAdminCred` to fire with `bool=true`.

## Mod file size budget

For reference, the production mods this knowledge bundle describes have these rough sizes:

| Mod | Lua source | Saved data |
|---|---|---|
| DinoStorage | About 2000 lines | Per-player JSON files, typically 4-10 KB each |
| BodyDrop | About 400 lines | None (stateless) |
| SkinMod | About 400 lines | Per-player JSON files, 500 bytes each |
| PlayerStats | About 200 lines | events.ndjson, grows over time, rotate daily |
| CommandBridge | About 500 lines | Inbox + results NDJSON, rotate daily |

UE4SS mods are not size-constrained in practice. Lua source files of 10000 lines load fine. The bottleneck (if any) is per-tick work, not source size.

## Closing notes

The single most important path to memorize is `<game>/TheIsle/Binaries/Win64/UE4SS.log`. That's where everything your mod logs goes. When something's not working, that's the first file to tail.

The second most important is `<game>/TheIsle/Binaries/Win64/Mods/mods.txt`. If your mod isn't loading, check it's enabled there.

The third is `<game>/TheIsle/Saved/Logs/TheIsle.log`. That's the engine's own log, useful for finding game-side errors that don't manifest in UE4SS.log.
