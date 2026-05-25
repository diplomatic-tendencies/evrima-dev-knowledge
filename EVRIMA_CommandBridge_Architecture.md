# CommandBridge architecture

CommandBridge is the IPC layer between a Discord bot (or any other external system) and the server-side Lua mods. It accepts NDJSON-formatted commands from the bot, routes them to the appropriate sub-mod's handler, and returns results back through a shared results file.

The purpose is to give bots a single entry point. Instead of every Lua mod opening its own listener and every bot integration touching multiple files, all command traffic goes through one ingress point with a routing key.

## File layout

```
Mods/CommandBridge/Saved/
├── inbox/
│   ├── DinoStorage.ndjson      # Commands targeted at DinoStorage
│   ├── BodyDrop.ndjson         # Commands targeted at BodyDrop
│   ├── SkinMod.ndjson          # Commands targeted at SkinMod
│   └── ...
├── results.ndjson              # Shared output stream
└── processed/                  # Optional: archive of processed commands
```

Each sub-mod gets its own inbox file. The bot writes a JSON line to the appropriate inbox; the sub-mod tails its inbox; the sub-mod writes its result to the shared `results.ndjson`.

The split-by-sub-mod inbox design avoids contention. If the bot writes 10 DinoStorage commands and 1 BodyDrop command in quick succession, the DinoStorage tail and the BodyDrop tail proceed independently. Neither blocks on the other.

The shared results file consolidates all results so the bot has a single output to consume.

## Command schema

Bot writes one line like:

```json
{"id":"abc-123","ts":1716508800,"cmd":"redeem","steam":"76561198XXXXXXX","args":{}}
```

Schema:

- `id` is a bot-generated correlation ID. The result line will echo it back so the bot can match request to response.
- `ts` is the bot-side timestamp, unix seconds. Used by the sub-mod to skip stale commands.
- `cmd` is the command name (interpreted by the sub-mod's handler).
- `steam` is the target steam ID (if the command acts on a player).
- `args` is a command-specific argument object.

The mod reads inbox lines, dispatches each to a registered sub-mod handler, and writes a result line:

```json
{"id":"abc-123","ts":1716508805,"status":"ok","cmd":"redeem","steam":"76561198XXXXXXX","detail":"redeemed Tyrannosaurus growth=1.0"}
```

Result schema:

- `id` echoes the request.
- `ts` is the result timestamp.
- `status` is `ok`, `error`, or `not-found`.
- `cmd` and `steam` echo the request for convenience.
- `detail` is a free-form message.

## Sub-mod routing

Each sub-mod registers a handler function with the bridge at boot:

```lua
-- In DinoStorage's main.lua, after presence registry init:

local function handleStoreCommand(payload)
    if payload.cmd == "store" then
        return doStore(payload.steam)
    elseif payload.cmd == "redeem" then
        return doRedeem(payload.steam)
    elseif payload.cmd == "info" then
        return getStoredInfo(payload.steam)
    end
    return { status = "error", detail = "unknown cmd" }
end

CommandBridge.register("DinoStorage", handleStoreCommand)
```

The bridge maintains a registry: `{ subModName -> handlerFn }`. It tails each registered inbox file and dispatches to the matching handler. If no handler is registered for a given inbox file, the bridge skips that file's tail.

The handler function returns a result object (status + detail), which the bridge serializes and writes to results.ndjson.

## Tail implementation

The bridge polls each inbox file every 500 milliseconds. For each file, it tracks the file position (last-read offset). New content gets read, split on newlines, and each line is dispatched.

```lua
local inboxPositions = {}  -- file -> last read offset

local function pollInbox(subMod, filePath, handler)
    local f, err = io.open(filePath, "r")
    if f == nil then return end

    local pos = inboxPositions[filePath] or 0
    f:seek("set", pos)

    local body = f:read("*a")
    if body == nil or #body == 0 then
        f:close()
        return
    end

    local newPos = f:seek()
    f:close()

    for line in body:gmatch("[^\n]+") do
        local ok, payload = pcall(decodeJson, line)
        if ok and payload ~= nil then
            local result = handler(payload)
            writeResult(payload, result)
        end
    end

    inboxPositions[filePath] = newPos
end

LoopInGameThreadWithDelay(500, function()
    for subMod, h in pairs(handlers) do
        local filePath = "Mods/CommandBridge/Saved/inbox/" .. subMod .. ".ndjson"
        pollInbox(subMod, filePath, h)
    end
end)
```

The file-position tracking is in-memory only. On server restart, the bridge starts from position zero of each inbox file, which means commands written before the restart are re-processed.

The fix for re-processing is the `cmd.flag` extension (described below) plus a timestamp check on the bot-side `ts` field. The bot's `ts` is compared against `os.time() - 60`; commands older than 60 seconds are skipped. This handles the restart-replay case cleanly: any command in flight when the server died is either too old to re-process (skipped) or current enough that re-processing is desirable.

## The `cmd.flag` extension

Some sub-mods (specifically DinoStorage) need additional flags on a per-command basis. The bridge accepts an optional `flag` field on the payload:

```json
{"id":"abc-123","cmd":"redeem","steam":"76561198XXXXXXX","args":{"flag":"force"}}
```

The flag is passed through to the handler as part of the args object. The handler interprets it. For example, DinoStorage's redeem handler recognizes `flag=force` to mean "redeem even if the species doesn't match what's stored."

## Results file management

The shared `results.ndjson` is append-only. Like PlayerStats' events.ndjson, the bot consumes by tailing. The same position-tracking pattern works:

```javascript
function pollResults() {
    // ...read new content from results.ndjson...
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

The bot maintains `pendingByCorrelationId` (request ID to callback function). When the bot sends a command, it generates an ID and stores a callback. When the matching result arrives, it invokes the callback. This is the standard request-response correlation pattern.

## Why files instead of a network socket

The file-based IPC is the simplest path that avoids three categories of problems:

First, it doesn't need a port. Network ports require firewall config, conflict checking, and a bind-listen-accept loop that's hard to do safely from Lua. Files are just files.

Second, it survives crashes on either side. If the bot dies mid-command, the file still exists; the server processes the command and writes a result that the bot will see when it restarts. If the server dies, the bot's pending commands stay in the inbox file; the server processes them on restart (with the timestamp filter to skip really stale ones).

Third, it's debuggable. You can `cat` the inbox files to see what commands are queued. You can `tail -f` the results to watch responses in real time. No need for special tooling.

The downside is latency: the 500-millisecond poll interval means commands take 500 milliseconds plus the handler's execution time to complete. For most use cases this is fine; for tight loops it isn't. The 500ms can be reduced to 100ms with minimal perf impact, but going lower than that increases the disk IO meaningfully without obvious benefit.

## Coordinating with sub-mods

Each sub-mod that wants to expose commands through the bridge needs to:

1. Call `CommandBridge.register(name, handlerFn)` at boot.
2. Implement the handler function as a synchronous function that returns a result object.
3. If the handler needs to do heavy work, queue the work in the handler and return an "accepted" result; do the actual work in a deferred tick.

The deferred-work pattern matters for heavy operations like state restore. The handler returns `{status="ok", detail="redeem queued"}` immediately; the actual mutation happens 3 seconds later in a separate tick. This keeps the bridge's poll loop fast and avoids any chance of the handler causing a hook-context crash (safety rule 5).

## Closing notes

The bridge is intentionally minimal. It does routing and serialization; the actual command logic lives in each sub-mod. The split keeps the bridge stable (rarely changes) while sub-mods can evolve their command surfaces freely.

A common addition is rate-limiting: per-player command rate limits to prevent bot bugs from spamming a single mod. The bridge can apply this generically by tracking command count per `(steam, sub-mod)` per minute, and rejecting commands above some threshold. This is a 30-line addition once you've decided on the policy.
