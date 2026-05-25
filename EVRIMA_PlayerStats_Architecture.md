# PlayerStats architecture

PlayerStats is a periodic snapshot emitter. Every 5 seconds it iterates online players via the presence registry, reads each player's vitals and basic state, and writes a single NDJSON line per snapshot to a tail-friendly events file. Bots and websites tail the file for live player-status displays.

The mod has no command surface. It's an event source for downstream consumers.

## Output format

One JSON object per line in `Mods/PlayerStats/Saved/events.ndjson`. Each line is a self-contained snapshot:

```json
{"ts":1716508800,"steam":"76561198XXXXXXX","name":"PlayerName","class":"BP_Tyrannosaurus_C","growth":1.0,"health":9500,"maxHealth":9500,"hunger":50,"maxHunger":100,"thirst":40,"maxThirst":100,"stamina":100,"maxStamina":100,"food":600,"maxFood":600,"blood":100,"oxygen":100,"isPrime":true,"isFemale":false,"location":{"x":12345,"y":67890,"z":100},"groupId":"abc123"}
```

Schema notes:

- `ts` is unix seconds, the snapshot wall-clock time.
- `steam` is the player's steam ID.
- `class` is the stripped class name (no path, no "BlueprintGeneratedClass" prefix).
- `groupId` may be empty string if the player is solo. Group semantics in `EVRIMA_Lua_Safety_Rules.md` are not relevant here; the GroupId comes from the per-player API.
- Location coordinates are in engine space (X, Y, Z). Note that the HUD displays Y as latitude and X as longitude; convert in your consumer if you need HUD-style display.

The file is append-only. Consumers tail it, parse new lines, and update their state.

The file grows roughly 200 bytes per online player per 5-second tick. A 100-player server produces about 1.5 MB per hour, 36 MB per day, 1 GB per month. Either rotate the file daily on the bot side (consume, then delete) or implement a rotation tick in the mod itself.

## Snapshot loop

```lua
local SNAPSHOT_INTERVAL_MS = 5000

local function snapshotOnce()
    local players = enumerateOnlinePlayers()
    local lines = {}
    local now = os.time()

    for _, p in ipairs(players) do
        if p.pawn ~= nil then  -- skip players on spawn-zone-select
            local row = buildSnapshotRow(p, now)
            if row ~= nil then
                lines[#lines+1] = encodeJson(row)
            end
        end
    end

    if #lines > 0 then
        appendLines("Mods/PlayerStats/Saved/events.ndjson", lines)
    end
end

LoopInGameThreadWithDelay(SNAPSHOT_INTERVAL_MS, snapshotOnce)
```

`enumerateOnlinePlayers` is the presence registry's helper (the `EVRIMA_Presence_Registry.md` doc covers it). The per-row builder reads each stat via the safe `GetX()` UFunctions and POD-struct field reads.

`buildSnapshotRow` is just a sequence of scalar reads:

```lua
local function buildSnapshotRow(p, ts)
    local pawn = p.pawn
    local row = {}
    row.ts = ts
    row.steam = p.steam

    pcall(function() row.name = p.controller:GetPlayerName():ToString() end)
    pcall(function()
        local cls = pawn:GetClass():GetFullName()
        row.class = stripBlueprintPrefix(cls)
    end)
    pcall(function() row.growth = pawn:GetGrowth() end)
    pcall(function() row.health = pawn:GetHealth() end)
    pcall(function() row.maxHealth = pawn:GetMaxHealth() end)
    pcall(function() row.hunger = pawn:GetHunger() end)
    -- ...etc for each stat...

    local loc
    pcall(function() loc = pawn:K2_GetActorLocation() end)
    if loc ~= nil then
        row.location = { x = loc.X, y = loc.Y, z = loc.Z }
    end

    return row
end
```

Every read is `pcall`-wrapped because any individual read could fail on a player in an edge state (mid-respawn pawn, ghost pawn, etc.). The defensive wrapping means one player in an edge state doesn't poison the whole snapshot batch.

## File append performance

A naive implementation that opens the file, writes one row, closes, repeats per player crushes disk performance. The right pattern is to accumulate the batch of rows in memory and write them all in one open-write-close cycle:

```lua
local function appendLines(path, lines)
    local f, err = io.open(path, "a")
    if f == nil then return false, err end
    for _, line in ipairs(lines) do
        f:write(line)
        f:write("\n")
    end
    f:close()
    return true
end
```

On the test rig, 100-player snapshot batches take under 5 milliseconds total. Not a perf concern at any reasonable server size.

## Why 5 seconds

The interval was chosen to balance freshness against load:

- Faster than 5 seconds: the snapshot data starts to feel real-time, but the file grows faster and the bot's poll loop has to be tighter. Diminishing returns for most consumer use cases.

- Slower than 5 seconds: the file is smaller but the bot's UI lags noticeably behind real game state. Players notice when the website shows their old growth value 30 seconds after they ate something.

5 seconds is the sweet spot for "live enough to feel current, sparse enough to not flood the consumer."

Some consumer use cases want different intervals. The mod's interval is a config knob; bot operators can request 2 seconds for a high-detail kill-feed integration or 30 seconds for a low-frequency player-list display.

## Coordination with other event sources

PlayerStats events.ndjson is one of several NDJSON streams that downstream bots consume. Other streams typically include:

- `Mods/KillFeed/Saved/events.ndjson` for kill events (if you're running KillFeed)
- `Mods/CommandBridge/Saved/results.ndjson` for command responses
- Whatever your custom mods emit

The bot's typical pattern is to tail every events.ndjson file independently, merge the streams by timestamp, and update its in-memory state model. PlayerStats provides the "what is everyone doing right now" baseline; the other streams provide the discrete events that change the baseline.

## Consumer-side tail pattern

A simple Node.js tail consumer:

```javascript
const fs = require('fs');
let position = 0;
let buffer = '';

function pollFile() {
    const stats = fs.statSync('events.ndjson');
    if (stats.size < position) {
        // File was rotated; reset
        position = 0;
        buffer = '';
    }
    if (stats.size > position) {
        const fd = fs.openSync('events.ndjson', 'r');
        const len = stats.size - position;
        const buf = Buffer.alloc(len);
        fs.readSync(fd, buf, 0, len, position);
        fs.closeSync(fd);
        position = stats.size;
        buffer += buf.toString('utf8');
    }
    let idx;
    while ((idx = buffer.indexOf('\n')) >= 0) {
        const line = buffer.slice(0, idx);
        buffer = buffer.slice(idx + 1);
        if (line.length > 0) {
            try {
                handleSnapshot(JSON.parse(line));
            } catch (e) {
                console.error('parse failed:', line.slice(0, 80));
            }
        }
    }
}

setInterval(pollFile, 1000);
```

The position-tracking pattern handles file rotation (size goes backwards) gracefully and processes any new appended data within a second of arrival.

## Closing notes

The mod is short, around 200 lines including the JSON encoder. The complexity is all in the things it does NOT do: it does not enumerate players via the broken APIs (uses the presence registry), it does not call methods on pawns that might be ghost pawns (uses livePawnFromCtrl), it does not open and close the file once per player.

The simplicity is the point. PlayerStats is a piece of infrastructure that other mods and external systems consume. The less surface area it has, the less likely it is to break.
