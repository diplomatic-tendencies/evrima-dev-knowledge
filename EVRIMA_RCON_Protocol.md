# EVRIMA RCON binary protocol

EVRIMA's RCON is not Source RCON. It is a custom binary protocol over TCP that uses single-byte command codes plus null-terminated text payloads.

This is the reference for that protocol, including the 13 known command codes (extracted from a working community bot's reverse-engineered code), the authentication flow, and notes on what's confirmed vs unconfirmed.

## Default ports

- Game traffic: 7777 (some servers move this to 7778)
- RCON traffic: 8888 (configurable in server config)

The RCON port is set via `Saved/Config/WindowsServer/Game.ini`:

```ini
[/Script/TheIsle.TIGameStateBase]
ServerRconEnabled=true
ServerRconPort=8888
ServerRconPassword=<password>
```

## Authentication flow

```
Client opens TCP socket to <server>:<rcon-port>
Client sends: 0x01 <password-bytes> 0x00
Server responds: 0x01 0x00 if accepted, 0x02 0x00 if rejected
```

The auth packet is a single byte 0x01 followed by the password as ASCII bytes, terminated by a single 0x00.

The server's response is also two bytes: the first byte is 0x01 for accepted or 0x02 for rejected, followed by 0x00.

After successful auth, the connection stays open. The client can send command bytes followed by their parameters; the server processes each command and may or may not send a response packet.

## Known command codes

The 13 command codes extracted from a working community bot's `rconConnector.js`:

| Byte | Command name | Parameter format | Notes |
|---|---|---|---|
| 0x01 | Authenticate | password + 0x00 | Initial handshake |
| 0x02 | Broadcast | text + 0x00 | Broadcast a message to all players |
| 0x03 | Announce | text + 0x00 | System-style announcement |
| 0x04 | Kick player | steamid + 0x00 reason + 0x00 | Kick by steam ID |
| 0x05 | Ban player | steamid + 0x00 duration + 0x00 reason + 0x00 | Ban with duration in seconds |
| 0x06 | List players | (no params) | Returns player list |
| 0x10 | Toggle whitelist | "true" or "false" + 0x00 | Enable/disable whitelist |
| 0x11 | Add to whitelist | steamid + 0x00 | |
| 0x12 | Remove from whitelist | steamid + 0x00 | |
| 0x13 | List whitelist | (no params) | Returns whitelist contents |
| 0x14 | Teleport player | steamid + 0x00 x + 0x00 y + 0x00 z + 0x00 | Coordinates as ASCII decimal |
| 0x15 | Save game state | (no params) | Trigger a save-now |
| 0x16 | Reload config | (no params) | Reload Game.ini changes |

All parameters are ASCII text terminated by 0x00. Numeric parameters (coordinates, durations) are formatted as decimal strings, not binary integers.

The byte values from 0x07 through 0x0F are unmapped in the bot's code. There may be additional commands the community bot did not implement; a small probe script that sends each unmapped byte and observes the response would enumerate them.

## Response format

Most commands return a response. The response format is the command byte echoed back, followed by null-terminated result text:

```
<command-byte> <result-text> 0x00
```

For the `List players` command (0x06), the response includes the full player list as a single text payload with newlines separating entries. Example response:

```
0x06 76561198XXXXXXX,PlayerName1,BP_Tyrannosaurus_C\n76561198YYYYYYY,PlayerName2,BP_Deer_C\n 0x00
```

The exact format varies by command. The community bot's code documents the expected format for each.

## Push events (unverified)

The community bot's code suggests the server may push unsolicited events on the RCON socket. Specifically, kill-feed events appear to be a candidate; the bot has code paths for receiving multi-byte packets with no preceding command.

This was NOT verified by direct testing. The probe approach (open the connection, sit idle, log all incoming bytes for an extended period during normal play) is the way to validate. If the server does push kill events, that's a significantly cheaper kill-feed source than the polling-based KillFeed mod design (since it would eliminate the need to instrument every damage call).

A probe script for this is documented in pseudocode:

```
open TCP to <server>:8888
send 0x01 <password> 0x00
read response, confirm 0x01 0x00
loop forever:
    read bytes (non-blocking with timeout)
    if bytes received: log hex + ASCII to file
    every 30 seconds: dump current buffer state
```

Run this during a session where players are killing each other on the server. Compare the timing of bytes received against known in-game kill events. Any consistent pattern that correlates with kill events would be the push-event format.

## Use cases

For external tooling, RCON is the cheapest path to:

- Broadcasting messages from a Discord bot to all players in-game.
- Kicking misbehaving players from a moderation tool.
- Whitelist management from an admin web UI.
- Save triggering before scheduled maintenance.
- Teleport coordination for events.

For features that need real-time observation (kill feeds, position tracking, chat reading), RCON is not sufficient. The push-event capability (if it exists) might cover kill feeds; otherwise the Lua-mod-side NDJSON streams (PlayerStats, KillFeed) are the path.

## Closing notes

The RCON protocol is well-suited for "external action on the server" but poorly suited for "observation of server state." Use it for commanding, not for watching.

Probe before relying on the unverified push-event behavior. The bot code that documented it may have been speculating; the actual on-the-wire behavior of the server is the source of truth.
