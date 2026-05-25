# EVRIMA RCON binary protocol

> Most opcodes here have been live-probed and behave as documented. Some argument formats and edge cases remain partially confirmed. Additional findings, corrections, and argument-format details welcome via issues or PRs.

EVRIMA's RCON is not Source RCON. It is a custom binary protocol over TCP that uses single-byte opcodes plus UTF-8 text payloads. This document reflects opcodes and frame structure that have been live-probed against a running server.

## Configuration

The RCON config lives in `Game.ini` under `[/Script/TheIsle.TIGameSession]`:

```ini
[/Script/TheIsle.TIGameSession]
bRconEnabled=true
RconPassword=<your password>
RconPort=8888
```

Default RCON port is 8888.

## Authentication

Frame format (client to server):

```
0x01 + <password bytes (UTF-8)>
```

The auth frame is a single byte `0x01` followed immediately by the password as plain UTF-8 bytes. No null terminator, no length prefix.

Server response on success: the literal string `Password Accepted`. **No trailing newline, no terminator byte.** The server sends the bytes and stops. Any read loop that waits for `\n` or `\n\n` on the auth response will hang indefinitely.

Working auth read strategy: read with a short timeout (~500 ms). When bytes arrive and the stream goes quiet, the response is complete.

```rust
async fn read_auth_response(stream: &mut TcpStream) -> Result<String, Box<dyn std::error::Error>> {
    let mut data = Vec::new();
    let mut buf = vec![0u8; 256];
    match tokio::time::timeout(std::time::Duration::from_millis(500), stream.read(&mut buf)).await {
        Ok(Ok(n)) if n > 0 => data.extend_from_slice(&buf[..n]),
        _ => {}
    }
    Ok(String::from_utf8_lossy(&data).trim().to_string())
}
```

## Command frame structure

All authenticated commands use the same frame:

```
0x02 + <opcode byte> + <arguments as UTF-8 bytes>
```

- The first byte `0x02` is the ExecCommand prefix. It is part of every command frame; it is not itself a command.
- The second byte is the opcode (see map below).
- Remaining bytes are arguments, comma-separated when multiple values are needed.
- No trailing null terminator is required. A working client sent commands without `0x00` terminators and all commands functioned.

Example (kick player frame, with steam ID `76561198XXXXXXX`):

```
02 30 37 36 35 36 31 31 39 38 58 58 58 58 58 58 58
│  │  └─────────────── ASCII bytes of the steam ID ──────────────┘
│  └ opcode 0x30 (KickPlayer)
└ 0x02 ExecCommand prefix
```

The server accepts this frame and logs it via `LogTheIsleCommandData`. Numeric arguments (steam IDs, coordinates, durations) are formatted as ASCII decimal text, not packed binary integers.

## Response termination

Three distinct termination behaviors. Conflating them causes hangs; separating them by command type resolves every hang.

| Response source | Termination | Notes |
|---|---|---|
| Auth | None | Server sends `Password Accepted` and stops. Read with timeout to detect end. |
| Player list (`0x40`) | `\n\n` | Response ends in a double newline. Empty server returns `PlayerList\n\n`. |
| Other commands | Single `\n` or nothing | Treat absence as "response complete." Reading until `\n\n` on these will hang. |

**Empty player list (verified hex):**

```
50 6c 61 79 65 72 4c 69 73 74 0a 0a   →  "PlayerList\n\n"
```

**Populated player list (verified hex fragment, one player):**

```
50 6c 61 79 65 72 4c 69 73 74 0a            "PlayerList\n"
37 36 35 36 31 31 39 38 ... 2c 0a           "<SteamID>,\n"
54 69 6e 79 6d 61 6e 7a 2c                  "<displayName>,"
```

## Verified opcode map

Every opcode below was confirmed by sending `0x02 <opcode>` and reading the server's `LogTheIsleCommandData` entry, which prints the command's internal name. Opcodes that produced a named log entry are confirmed.

| Opcode | Server-logged name | Args (where verified) |
|---|---|---|
| `0x01` | (Auth)¹ | password (no `0x02` prefix on auth) |
| `0x10` | Announce | message |
| `0x11` | DirectMessage | `SteamID,message` |
| `0x12` | ServerDetails | (none) |
| `0x13` | WipeCorpses | (none) |
| `0x14` | GetPlayables | (none) |
| `0x15` | UpdatePlayables² | (none) |
| `0x19` | ToggleMigrations² | (none) |
| `0x1A` | AddPlayable | class name |
| `0x1B` | RemovePlayable | class name |
| `0x20` | BanPlayer | `Name,SteamID64,Reason,Time` |
| `0x21` | ToggleGrowthMultiplier² | (none) |
| `0x22` | SetGrowthMultiplier² | `SteamID,value` |
| `0x23` | ToggleNetUpdateDistanceChecks² | (none) |
| `0x30` | KickPlayer | `SteamID64` |
| `0x40` | GetPlayerList | (none) |
| `0x50` | Save | (none) |
| `0x60` | Pause³ | (none) |
| `0x70` | Command | (free-form Unreal-style command) |
| `0x77` | GetPlayerData | (none; returns full per-player data) |
| `0x81` | ToggleWhitelist | (none) |
| `0x82` | AddWhitelistIDs | `SteamID(s)` (comma-separated) |
| `0x83` | RemoveWhitelistIDs | `SteamID(s)` (comma-separated) |
| `0x84` | ToggleGlobalChat | (none) |
| `0x86` | ToggleHumans | (none) |
| `0x90` | ToggleAI | (none) |
| `0x91` | DisableAIClasses | class list (comma-separated) |
| `0x92` | AdjustAIDensity | value |
| `0x93` | GetQueueStatus | (none) |
| `0x94` | ToggleAILearning | (none) |

**Footnotes:**

1. Auth uses opcode `0x01` directly, NOT the `0x02` ExecCommand prefix. It is its own frame type.
2. These opcodes appeared as named entries in the probe log. Their exact in-game effect and full argument format beyond what is shown was not independently verified.
3. `0x60 Pause` appears in the probe log (the opcode name exists in the server's command table) but multiple users report it times out with no observable in-game effect. Likely a stub or broken in this server version. If you can get it to actually pause the world, please open an issue with the working invocation.

## Verified absence: no slay opcode

A full sweep of the entire `0x00`-`0xFF` range produced no opcode named "Slay" or any kill/death-trigger equivalent. **Slaying a dinosaur is not exposed via RCON in this server version.** The in-game admin panel has a slay function; RCON does not.

For features that need to kill or damage a player (admin punishment, scripted events), the path is server-side Lua (`SetHealth(0)` on the target pawn). See the BodyDrop architecture doc for an example of this pattern.

## Argument format notes

### Kick (`0x30`)

```
<SteamID64>
```

Frame accepted and logged. Note: the server does not allow an admin to kick their own connected character; the frame is accepted but produces no effect when self-targeted.

### Ban (`0x20`)

```
Name,SteamID64,Reason,Time
```

Four comma-joined fields. The `Time` field's units (seconds vs minutes vs another unit) were not independently verified.

### Direct message (`0x11`)

```
SteamID64,message
```

The message is plain text; commas in the message body may interact with the parser. Verified format with a payload like `76561198XXXXXXX,hello`.

### Whitelist add and remove (`0x82` and `0x83`)

```
<SteamID64>
```

Single or multiple steam IDs, comma-separated when supplying multiple. The server logs the IDs back.

## Probe methodology

The opcode map above was generated by:

1. Authenticate via `0x01 <password>` and confirm response.
2. For each opcode value from `0x00` to `0xFF`:
   - Send the frame `0x02 <opcode>` with no arguments.
   - Read response with timeout.
   - Inspect the server log for `LogTheIsleCommandData` entries matching the opcode.
3. Opcodes that produced a named log entry are recorded; opcodes that produced silence or a generic "unknown command" log are omitted.

Run this against a test server, not a production server with players online. Many commands have side effects (toggle whitelist, toggle global chat, etc.) and probing them blindly will disturb live play.

## Use cases

For external tooling, RCON is the cheapest path to:

- Broadcasting messages to all players (`0x10 Announce`).
- Direct-messaging specific players (`0x11 DirectMessage`).
- Kicking and banning (`0x30`, `0x20`).
- Whitelist management (`0x81`-`0x83`).
- Save triggering before maintenance (`0x50`).
- Reading the player list and per-player data (`0x40`, `0x77`).
- Toggling AI features (`0x90`-`0x94`).

For features that need real-time event observation (kill feeds, position tracking, chat reading), RCON is not sufficient. The Lua-mod-side NDJSON streams (PlayerStats, KillFeed) are the path for that.

## Closing notes

The RCON protocol is well-suited for "external action on the server" but limited for "observation of server state." Use it for commanding plus periodic queries; use Lua-mod-side NDJSON streams for real-time event observation.

The frame structure (the `0x02` ExecCommand prefix) is the single most important detail to get right. Sending bare opcodes without the `0x02` prefix produces silent rejection or unexpected behavior on the server side.

## Credits

Live opcode probe across the full `0x00`-`0xFF` range, verified frame structure, response-termination behaviors, and ongoing opcode confirmations by **MehO** and **Tinymanz**.
