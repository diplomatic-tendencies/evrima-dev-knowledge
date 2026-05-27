# CommandBridge architecture

CommandBridge is the IPC layer between a Discord bot (or any other external system) and the server-side Lua mods. It tails a single NDJSON command file, dispatches each line to a built-in verb handler or routes it into a sub-mod's own inbox, and writes results back to a shared file.

The purpose is to give bots a single entry point. Instead of every Lua mod opening its own listener and every bot integration touching multiple files, all command traffic goes through one ingress point with a verb name.

## File layout

```
Mods/CommandBridge/Saved/
├── commands.ndjson         # Single command input file (bot writes here)
├── results.ndjson          # Shared output stream (mods + bridge write, bot reads)
├── config.json             # Bridge config (poll interval, input mode, etc.)
└── reload.flag             # Hot-reload trigger (optional)

Mods/<SubMod>/Saved/
└── inbox.ndjson            # Per-sub-mod inbox (BodyDrop, PlayerStats, ...)

Mods/DinoStorage/Saved/
└── cmd.flag                # DinoStorage's legacy command file (NOT NDJSON)
```

CommandBridge polls a **single** `commands.ndjson` (NOT a per-sub-mod directory under CommandBridge). Sub-mod inboxes live in each sub-mod's own `Saved/` directory; CommandBridge writes commands into them via the routing handlers.

The shared `results.ndjson` consolidates results from the bridge itself, from each sub-mod that runs commands, and from DinoStorage's legacy `cmd.flag` path. The bot has one output file to consume.

## Command schema

Bot writes one line like:

```json
{"id":"uuid","ts":1779000000,"verb":"prime","steam":"76561198XXX","args":{}}
```

```json
{"id":"uuid","ts":1779000000,"verb":"skin","steam":"76561198XXX","args":{"customizer":{"BodyColor":{"R":0.8,"G":0.2,"B":0.1}}}}
```

```json
{"id":"uuid","ts":1779000000,"verb":"teleport","steam":"76561198XXX","args":{"x":12345,"y":67890,"z":22500}}
```

Schema:

- `id` is a bot-generated correlation ID. Result lines echo it back so the bot can match request to response.
- `ts` is the bot-side timestamp, unix seconds. Currently informational — the bridge does not apply staleness filtering on it.
- `verb` is the command name (NOT `cmd`).
- `steam` is the target steam ID (if the verb acts on a player).
- `args` is a verb-specific argument object. May contain nested objects (e.g. `args.customizer.BodyColor.R`); the parser uses `%b{}` balanced-brace matching so nested objects don't truncate.

The bridge writes a result line for each dispatched verb:

```json
{"id":"uuid","ts":1779000001,"verb":"prime","steam":"76561198XXX","ok":true,"msg":"prime forced; pick slot 4 mutation in-game"}
```

Result schema (live `emitResult`):

- `id` echoes the request.
- `ts` is the result timestamp (server-side unix seconds).
- `verb` echoes the request verb.
- `steam` echoes the request steam.
- `ok` is a bool.
- `msg` is a free-form message.

Sub-mod results add `source` (the mod's name, e.g. `"BodyDrop"`) plus `args` (the tokens or array the sub-mod received) on top of the bridge's base shape. Bots can correlate the request → ACK → final-result chain via `id`, and distinguish the bridge ACK from the sub-mod result by checking for the `source` field's presence.

## Direct verb handlers

These are implemented inside CommandBridge itself and dispatch synchronously against the player's pawn:

| Verb | What it does | Args |
|---|---|---|
| `prime` | Force prime eligibility on a player (all 10 conditions + cached bool) | (none beyond `steam`) |
| `unprime` | Clear prime eligibility | (none) |
| `skin` | Set color slot(s) on the player's customizer | Two modes — see below |
| `mutations` | Set mutation slot names (FName) | `args.slot1`..`args.slot4` OR `Slot1`..`Slot4` OR `MutationSlot1`..`MutationSlot4`. **Handler MUST wrap values in `FName(v)` before writing to the struct field** — raw Lua strings into FName fields violate Safety Rule 2 and will crash UE4SS. Verified live 2026-05-26. |
| `teleport` | Move player to engine coords | `args.{x,y,z}` plus optional `args.yaw` (used by the K2_TeleportTo fallback path). |
| `kill` | Force player death | (none) |
| `heal` | Restore HP to max | (none) |
| `setgrowth` | Set growth (0.0–1.0) | `args.value` |
| `setvital` | Set a single vital | `args.{name,value}` where name is `hunger`/`thirst`/`stamina`/etc. |
| `notify` | Show client-side notification text | `args.message`. Note: `ClientShowNotification` is intermittently flaky in production, especially right after a player spawns. The handler honestly returns `ok=false` when the call errors, but transient failures will leave the player without their popup. |

### `skin` verb dual mode

**Single-field mode** (good for simple one-off changes):

```json
{"verb":"skin","steam":"76561198XXX","args":{"field":"body","r":1.0,"g":0.5,"b":0.0}}
```

`args.field` is one of `body | markings | flank | underbelly | detail | eyes | breed | all`. `all` applies the RGB to every visible field. Alpha is hardcoded to 1.0.

**BULK mode** (good for website/bot integrations setting the full customizer atomically):

```json
{"verb":"skin","steam":"76561198XXX","args":{"customizer":{
  "BodyColor":       {"r":0.8, "g":0.2, "b":0.1, "a":1.0},
  "MarkingsColor":   {"r":0.2, "g":0.2, "b":0.2, "a":1.0},
  "FlankColor":      {"r":0.5, "g":0.3, "b":0.1, "a":1.0},
  "UnderbellyColor": {"r":1.0, "g":1.0, "b":1.0, "a":1.0},
  "Detail1Color":    {"r":0.0, "g":0.0, "b":0.0, "a":1.0},
  "EyesColor":       {"r":1.0, "g":0.0, "b":0.0, "a":1.0},
  "MaleDisplayColor":{"r":0.0, "g":1.0, "b":0.0, "a":1.0},
  "SkinVariation": 0.5,
  "PatternIndex": 1
}}}
```

BULK mode is the recommended path for website/bot integrations — one atomic SetCustomizerData call replicates without visual flicker.

**Color sub-keys must be lowercase `r/g/b/a`** matching the convention used by PlayerStats events and SkinMod's save file. (Some published versions of the handler check uppercase `R/G/B/A` only — if you fork from a copy that pre-dates the 2026-05-26 fix, you'll see colors silently drop while the response claims success. Verify by reading the handler's `src.R or src.r` fallback.)

BULK mode does NOT write `bIsFemale` even if present in the payload — gender change requires a respawn flow, not a customizer update.

The verb-to-handler table is hard-coded in CommandBridge's `handlers` table. There is no public `register()` API. Adding a new direct verb means editing CommandBridge itself; routing to a sub-mod is the alternative (see below).

## Sub-mod routing

For verbs handled by a separate mod, CommandBridge has routing handlers that **write the command into that sub-mod's own inbox file** at `Mods/<SubMod>/Saved/inbox.ndjson`:

| Verb | Routes to | Inbox path |
|---|---|---|
| `bd`, `bodydrop` | BodyDrop | `Mods/BodyDrop/Saved/inbox.ndjson` |
| `ps`, `playerstats` | PlayerStats | `Mods/PlayerStats/Saved/inbox.ndjson` |
| `dino_store`, `dino_redeem`, `dino_delete`, `dino_list` | DinoStorage | `Mods/DinoStorage/Saved/cmd.flag` (legacy format, see below) |
| `dino_setmax`, `dino_clearmax`, `dino_getmax` | DinoStorage | Same `cmd.flag` (per-player slot-cap CRUD) |

Each sub-mod is responsible for tailing its own inbox file. Each runs a `pollInbox` loop that processes new lines, executes the command, and writes a result to the shared `Mods/CommandBridge/Saved/results.ndjson`.

The bot sees:
1. CommandBridge's immediate "queued" ACK with the original id (no `source` field).
2. The sub-mod's actual result with the same id (includes `source` field naming the sub-mod).

### Sub-mod inbox tailer pattern

BodyDrop and PlayerStats implement this identically:

```lua
local INBOX_PATH     = SAVED_DIR .. "/inbox.ndjson"
local SHARED_RESULTS = "Mods/CommandBridge/Saved/results.ndjson"

local function pollInbox()
    if not fileExists(INBOX_PATH) then return end
    -- Atomic-ish rename so the bot can keep writing while we process
    local stash = INBOX_PATH .. ".processing"
    os.remove(stash)
    os.rename(INBOX_PATH, stash)
    local body = readAll(stash)
    if body == nil or body == "" then os.remove(stash); return end

    for line in body:gmatch("[^\n]+") do
        local id    = jsonReadString(line, "id")
        local steam = jsonReadString(line, "steam") or ""
        local argsBlock = string.match(line, '"args"%s*:%s*%[([^%]]*)%]')
        local args = {}
        if argsBlock then
            for s in string.gmatch(argsBlock, '"([^"]*)"') do
                args[#args+1] = s
            end
        end

        local ok, msg = pcall(function() return handleCommand(steam, args) end)
        if not ok then msg = "error: " .. tostring(msg) end

        appendLine(SHARED_RESULTS, buildResultLine(id, steam, args, ok, msg))
    end
    os.remove(stash)
end
```

The mod's existing `handleCommand(steam, args)` chat dispatch IS the inbox handler — no new logic, just exposed via file.

## DinoStorage legacy `cmd.flag`

DinoStorage predates this NDJSON pattern and uses a simpler newline-separated `verb arg` format. The bridge accommodates by writing into `Mods/DinoStorage/Saved/cmd.flag`. Optional `[id]` prefix lets results correlate:

```
[abc123] store 76561198XXXXXXXXX
[abc124] redeem 76561198YYYYYYYYY 1
```

DinoStorage parses the id prefix and writes its result to the same `Mods/CommandBridge/Saved/results.ndjson`. Backward compatible — lines without the id prefix still work, they just don't correlate.

## Bridge tail implementation

```lua
local POLL_INTERVAL_MS = 1000   -- fixed loop cadence

LoopInGameThreadWithDelay(POLL_INTERVAL_MS, function()
    safeCall("pollInput", pollInput)
    -- ... reload-flag handling ...
end)

local lastPollAt = 0
local function pollInput()
    if not config.enabled then return end
    local now = os.time()
    if (now - lastPollAt) < config.inputPollSeconds then return end
    lastPollAt = now

    -- Read commands.ndjson (or fetch from inputUrl in http mode), parse each
    -- line via parseCommand(), dispatch via the verb handler table.
    -- ...
end
```

The outer loop fires every 1000 ms. The inner `pollInput` is gated by `config.inputPollSeconds` (default `2` per the live config defaults; lower for tighter latency).

The bridge supports two input modes:

- `inputMode = "file"` (default): read `Mods/CommandBridge/Saved/commands.ndjson`. Atomic-ish rename to `.processing` while parsing so the bot can keep writing.
- `inputMode = "http"`: GET `config.inputUrl` (with optional `inputAuthHeader`), parse the NDJSON body. The bot must mark consumed commands on its end.

## Bot-side result consumption

Bot maintains `pendingByCorrelationId` (request ID to callback function). When the bot sends a command, it generates an ID and stores a callback. When the matching result arrives in `results.ndjson`, it invokes the callback:

```javascript
function pollResults() {
    for (const line of newLines) {
        const result = JSON.parse(line);
        const callback = pendingByCorrelationId[result.id];
        if (callback) {
            callback(result);
            delete pendingByCorrelationId[result.id];
        }
    }
}
```

Standard request-response correlation. Note that the bot may receive TWO result lines per command if the verb routes to a sub-mod (one ACK from the bridge, one final from the sub-mod). The bridge's ACK lacks the `source` field while the sub-mod's final result includes `source: "BodyDrop"` (or similar). Bots that want the final result should wait for the line WITH a `source` field; bots that want the fastest ACK can act on the first line.

## Why files instead of a network socket

The file-based IPC is the simplest path that avoids three categories of problems:

First, it doesn't need a port. Network ports require firewall config, conflict checking, and a bind-listen-accept loop that's hard to do safely from Lua.

Second, it survives crashes on either side. If the bot dies mid-command, the file still exists; the server processes the command and writes a result that the bot will see when it restarts. If the server dies, the bot's pending commands stay in the inbox file; the server processes them on restart.

Third, it's debuggable. You can `cat` the inbox files to see what commands are queued. You can `tail -f` the results to watch responses in real time. No special tooling needed.

The downside is latency: with the default 2-second `inputPollSeconds` plus the sub-mod's own 1–3 second poll, routed commands take roughly 3–5 seconds end-to-end. Direct verbs return on the bridge's own 1-second outer loop after the inputPollSeconds gate. Tighten `inputPollSeconds` to 1 (or even 0) for development; the disk IO impact is negligible at typical command rates.

## Adding a new sub-mod

To wire a new mod through the bridge:

1. The sub-mod implements its own `pollInbox` loop reading `Mods/<YourMod>/Saved/inbox.ndjson`.
2. The sub-mod writes results to `Mods/CommandBridge/Saved/results.ndjson`.
3. Edit CommandBridge to add a routing handler:

```lua
handlers.yourverb = function(steam, args, cmdId)
    local tokens = extractTokens(args)
    return writeToInbox("YourMod", cmdId, steam, tokens)
end
```

There is no plugin/auto-discovery mechanism; CommandBridge needs the routing entry to know where to send the command. This is intentional — explicit routing keeps the bridge stable and predictable.

## Closing notes

The bridge is intentionally minimal. It does input parsing, verb dispatch, sub-mod routing, and result aggregation; the actual command logic lives in CommandBridge itself (for direct verbs) or in each sub-mod (for routed verbs). The split keeps the bridge stable while sub-mods can evolve their command surfaces freely.

A common consideration is rate-limiting: per-player or per-source command rate limits to prevent bot bugs from spamming a single mod. The bridge does not currently enforce this; add it in the dispatch loop if needed, with a per-`(steam, verb)` count per minute and a reject branch above some threshold.
