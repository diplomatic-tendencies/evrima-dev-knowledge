# PlayerStats architecture

PlayerStats is a periodic snapshot emitter. Every 5 seconds (configurable via `emitIntervalSeconds`) it iterates online players via the presence registry, reads each player's vitals and state, and writes a nested NDJSON line per snapshot to a tail-friendly events file. Bots and websites tail the file for live player-status displays.

The mod also exposes an admin chat command surface (`!playerstats`, `!ps`) and a CommandBridge inbox handler for runtime control. Both are documented below.

## Output format

One JSON object per line in `Mods/PlayerStats/Saved/events.ndjson`. Each line is a self-contained nested snapshot. Real shape (matching the live mod's `capturePlayerSnapshot`):

```json
{
  "ts": 1716508800,
  "steam": "76561198XXXXXXX",
  "species": "/Game/TheIsle/Core/Characters/Dinosaurs/Tyrannosaurus/BP_Tyrannosaurus.BP_Tyrannosaurus_C",
  "growth": 1.0,
  "pos": { "x": 12345.0, "y": 67890.0, "z": 100.0 },
  "vitals": {
    "hp": 9350.0, "hpMax": 9350.0,
    "hunger": 50.0, "hungerMax": 100.0,
    "thirst": 40.0, "thirstMax": 1000.0,
    "stamina": 778.0, "staminaMax": 778.0,
    "food": 600.0, "foodMax": 600.0,
    "oxygen": 776.0, "blood": 9350.0,
    "lockedDamage": 0.0, "rottenValue": 1800.0, "waterLevel": -1022646.93
  },
  "nutrients": {
    "carb": 0.0, "protein": 0.0, "lipid": 0.0, "bones": 0.0,
    "cannibal": 0.0, "magy": 0.0, "rottenFlesh": 0.0, "mushrooms": 0.0,
    "malnutrition": false
  },
  "mutations": { "MutationSlot1": "Truculency" },
  "prime": { "eligible": true, "conditions": [true, false, ...], "metCount": 5 },
  "skin": {
    "body":       { "r": 0.5, "g": 0.5, "b": 0.5, "a": 1.0 },
    "markings":   { "r": 0.3, "g": 0.3, "b": 0.3, "a": 1.0 },
    "flank":      { "r": 0.8, "g": 0.8, "b": 0.8, "a": 1.0 },
    "underbelly": { "r": 0.1, "g": 0.1, "b": 0.1, "a": 1.0 },
    "detail1":    { "r": 0.5, "g": 0.5, "b": 0.5, "a": 1.0 },
    "eyes":       { "r": 0.4, "g": 0.4, "b": 0.4, "a": 1.0 },
    "maleDisplay":{ "r": 0.6, "g": 0.6, "b": 0.6, "a": 1.0 },
    "skinVariation": 0.5,
    "patternIndex": 0
  }
}
```

Schema notes:

- `ts` is unix seconds, the snapshot wall-clock time.
- `steam` is the player's steam ID.
- `species` (NOT `class`) is the full class path of the player's dino.
- `pos` (NOT `location`) is engine-space `{x, y, z}`. Note the HUD displays Y as latitude and X as longitude; convert in your consumer if you need HUD-style display.
- `vitals.hp`/`hpMax` (NOT `health`/`maxHealth`) — naming is intentional, matches DinoStorage's storage schema for round-trip consistency.
- `prime.conditions` is the 10-bool array; `prime.eligible` is the cached output; `prime.metCount` is the int count of true conditions.
- `skin` uses friendly aliases (`body`, `markings`, `flank`, etc.) with lowercase `r/g/b/a` color sub-keys. See `EVRIMA_Customizer_Field_Map.md` for the underlying engine field names.
- `nutrients` keys are friendly aliases (`carb`, `protein`, etc.), not the underlying struct names (`CarbValue`, `ProteinValue`).
- `mutations` is a sub-object with only the populated slot fields (engine names like `MutationSlot1`).

**Fields NOT emitted** that one might expect:
- `name` / player name — not in the row.
- `isFemale` — not in the row; check `species` and the in-game UI.
- `groupId` — not in the row. The underlying `TICharacterBase.GroupId` field exists on the pawn (every player has a unique non-zero int; group members share a value), but PlayerStats does not emit it. BodyDrop's `groupPolicy` filter reads it directly from each pawn instead.

The file is append-only. Consumers tail it, parse new lines, and update their state.

The file grows roughly 600 bytes per online player per 5-second tick (nested format is larger than the flat shape an earlier version of this doc described). A 100-player server produces about 4.5 MB per hour, 100 MB per day. The mod includes built-in rotation — see "File rotation" below.

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

`capturePlayerSnapshot` is a sequence of nested reads. Each section is `pcall`-wrapped because any individual read could fail on a player in an edge state (mid-respawn pawn, ghost pawn, etc.). The defensive wrapping means one player in an edge state doesn't poison the whole snapshot batch.

```lua
local function capturePlayerSnapshot(p, ts)
    local pawn = p.pawn
    local row = { ts = ts, steam = p.steam }

    pcall(function()
        local cls = pawn:GetClass():GetFullName()
        row.species = stripClassPrefix(cls)
    end)
    pcall(function() row.growth = pawn:GetGrowth() end)

    local loc
    pcall(function() loc = pawn:K2_GetActorLocation() end)
    if loc ~= nil then row.pos = { x = loc.X, y = loc.Y, z = loc.Z } end

    -- Nested vitals (note: hp/hpMax, NOT health/maxHealth)
    row.vitals = {}
    pcall(function() row.vitals.hp = pawn:GetHealth() end)
    pcall(function() row.vitals.hpMax = pawn:GetMaxHealth() end)
    pcall(function() row.vitals.hunger = pawn:GetHunger() end)
    pcall(function() row.vitals.hungerMax = pawn:GetMaxHunger() end)
    -- ...etc...

    -- Nested skin block (friendly aliases + lowercase rgba)
    pcall(function()
        local cdata = pawn:GetCustomizerData()
        if cdata ~= nil then
            row.skin = {
                body       = readColor(cdata, "BodyColor"),
                markings   = readColor(cdata, "MarkingsColor"),
                flank      = readColor(cdata, "FlankColor"),
                underbelly = readColor(cdata, "UnderbellyColor"),
                detail1    = readColor(cdata, "Detail1Color"),
                eyes       = readColor(cdata, "EyesColor"),
                maleDisplay= readColor(cdata, "MaleDisplayColor"),
                skinVariation = cdata.SkinVariation,
                patternIndex  = cdata.PatternIndex,
            }
        end
    end)

    -- ...nutrients, mutations, prime blocks similar...
    return row
end

local function readColor(cdata, fieldName)
    local c = cdata[fieldName]; if c == nil then return nil end
    return { r = c.R, g = c.G, b = c.B, a = c.A or 1.0 }
end
```

**0.21.720 note on the skin block:** `GetCustomizerData()` is suspect since the skin overhaul (the write-side wrapper on the same surface silently broke, and the read wrapper is no longer trusted — see [EVRIMA_Customizer_Field_Map.md](EVRIMA_Customizer_Field_Map.md)). New captures should read the live `pawn.CustomizerData` property instead, which also carries the three new color regions (teeth, mouth, claws) the patch added. The wrapper shape above is kept as the verified pre-patch form.

## File append + rotation

Lines are appended one-at-a-time via `appendLine`. Rotation is built in via `rotateIfNeeded`:

```lua
-- Defaults (overridable via Saved/config.json):
local FILE_ROTATE_MAX_BYTES = 52428800   -- 50 MB
local ROTATION_MAX_AGE_SEC  = 86400      -- 24 h

local function rotateIfNeeded(path)
    local size = fileSize(path)
    if size == nil or size < config.fileRotateMaxBytes then return end
    local rotated = path .. "." .. os.time() .. ".old"
    os.rename(path, rotated)
    pruneRotations(path)
end

local function pruneRotations(basePath)
    -- Windows: dir /B + parse mtime via Lua-side stat, delete .old files
    -- older than rotationMaxAgeSec.
end
```

When the file exceeds `fileRotateMaxBytes` (default 50 MB), the mod renames it to `events.ndjson.<unixTs>.old` and starts a fresh `events.ndjson`. `pruneRotations` later deletes `.old` files older than `rotationMaxAgeSec` (default 24 h). The pruning logic uses Windows `dir /B` shell-out (Windows-only — the dedicated server is Windows-only anyway).

Consumer tail loops should be robust to file rotation: if the file's reported size goes DOWN between polls, reset their position offset to 0 and continue. The Node.js consumer example below shows the pattern.

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

## Chat command surface

PlayerStats exposes admin chat commands (`!playerstats`, `!ps`) plus a CommandBridge inbox handler. Subcommands:

| Command | Behavior |
|---|---|
| `!ps status` | Returns one-line status: `PlayerStats v001 \| ON \| interval=5s mode=file out=Mods/PlayerStats/Saved/events.ndjson` |
| `!ps on` | Enable emission |
| `!ps off` | Pause emission |
| `!ps force` | Force one snapshot immediately |
| `!ps tick` | Manually trigger one tick |
| `!ps reload` | Reload config from `Saved/config.json` |
| `!ps test` | Internal diagnostic |

Both chat and inbox paths are admin-gated via `config.adminSteamIds`. Non-admins get a reply but no state change.

The CommandBridge inbox lives at `Mods/PlayerStats/Saved/inbox.ndjson` and accepts the same subcommand tokens (`["status"]`, `["force"]`, etc.) via the standard sub-mod inbox pattern documented in `EVRIMA_CommandBridge_Architecture.md`.

## Closing notes

The mod is short, around 830 lines including the rotation logic and chat surface. The complexity is mostly in the things it does NOT do: it does not enumerate players via the broken APIs (uses the presence registry), it does not call methods on pawns that might be ghost pawns (uses livePawnFromCtrl), it does not invent file IO patterns where simple per-line append + size-based rotation works.

The simplicity is the point. PlayerStats is a piece of infrastructure that other mods and external systems consume. The less surface area it has, the less likely it is to break.
