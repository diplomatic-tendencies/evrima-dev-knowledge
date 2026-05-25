# UE4SS Lua mod hot-reload mechanism

Short technical guide for adding "live mod reload without server restart" to UE4SS Lua mods. Verified working on The Isle EVRIMA dedicated server.

## What this actually is

This is **per-mod state teardown and re-init via UE4SS's `RestartCurrentMod()` API**, triggered by an external file flag. NOT true Lua-code patching where running functions get swapped in-place.

What that means in practice:
- Faster than a server restart (sub-second vs 30-60s)
- Doesn't kick players, doesn't wipe world state
- Re-reads `main.lua` from disk, re-runs it from top
- All in-memory state of THIS mod is lost; other mods unaffected

That distinction matters when you're explaining what your mod does.

## The implementation (~15 lines)

```lua
local RELOAD_FLAG = "Mods/YourMod/Saved/reload.flag"

-- Reads a flag file. Returns content (trimmed), nil if absent.
-- Deletes the file on consumption so reload doesn't fire every tick.
local function consumeFlag(path)
    local f = io.open(path, "rb"); if f == nil then return nil end
    local body = f:read("*all"); f:close(); os.remove(path)
    if body == nil then return nil end
    body = body:gsub("^%s+", ""):gsub("%s+$", "")
    if body == "" then return nil end
    return body
end

-- In your mod's main poll loop:
LoopInGameThreadWithDelay(POLL_INTERVAL_MS, function()
    -- ...your other periodic work...

    local reload = consumeFlag(RELOAD_FLAG)
    if reload ~= nil and RestartCurrentMod ~= nil then
        print(string.format("[YourMod] RELOAD token=%s\n", reload))
        RestartCurrentMod()
    end
end)
```

That's the whole pattern. `RestartCurrentMod` is a UE4SS-provided global. The `~= nil` check is defensive because some UE4SS builds don't expose it.

## How to trigger a reload

Drop a file at `Mods/YourMod/Saved/reload.flag` with any non-empty content.

PowerShell:
```powershell
"manual" | Out-File -Encoding ascii "C:\path\to\server\TheIsle\Binaries\Win64\Mods\YourMod\Saved\reload.flag"
```

Node.js:
```javascript
const fs = require('fs');
const path = require('path');
fs.writeFileSync(
    path.join(modSavedDir, 'reload.flag'),
    Date.now().toString()
);
```

Bash:
```bash
echo "manual" > "/path/to/Mods/YourMod/Saved/reload.flag"
```

The flag body content becomes a "token" used for logging. SkinMod logs `RELOAD token=<value>` so you can tell which reload event fired. Write anything: a timestamp, a version string, the literal "manual", or just a single space.

## What survives vs what gets wiped

**Wiped on reload:**
- All `local` variables at module scope
- Tables tracking spawned actors, players, pending commands
- Cached state (presence registry, command-dedup table, observation queues)
- Active `LoopInGameThreadWithDelay` timers from the previous instance
- Hook references may double-fire briefly during the changeover window

**Survives:**
- Files in your mod's `Saved/` directory (use this for persistence across reloads)
- Spawned actors in the world (corpses, AI dinos, nests etc. don't care that the mod that spawned them reloaded)
- Players' in-game state (vitals, growth, customizer colors - that lives on their pawns, not on your mod)
- Other mods' Lua state (they're separate VMs)
- Server connection (no players are kicked)

## Practical implications

- **Track in `Saved/` if you need it after reload.** Anything in module-scope `local` tables is gone.
- **Hook registration is idempotent in UE4SS most of the time.** If your `RegisterHook` calls run twice (once before reload, once after), UE4SS usually dedupes. Not guaranteed in edge cases.
- **Brief command-loss window.** Chat commands sent during the sub-second reload tick may be missed. Production-safe mods often queue critical commands to disk via NDJSON to survive this.
- **The flag-poll interval gates your minimum reload latency.** If your mod polls every 2 seconds, reloads happen with up to 2s delay after you drop the flag.

## When to use it vs when to do a full restart

| Use reload.flag when | Use full server restart when |
|---|---|
| Tuning config values during dev | Changing which UFunctions you hook |
| Iterating on chat-command logic | Removing or renaming existing hooks |
| Hot-fixing a bug in production | Mod has large in-memory state you can't afford to lose |
| Bot-delivered mod updates | Multiple mods need coordinated update |
| The mod's tracking state is recoverable | Anything safety-critical where a brief hook-double-fire is unacceptable |

## Speed comparison

| Operation | Time | Players affected | World state |
|---|---:|---|---|
| `reload.flag` for one mod | < 1 second | none | preserved |
| Single mod restart (this) | < 1 second | none | preserved |
| `mods.txt` toggle + UE4SS reload | seconds | none directly | preserved |
| Full server process restart | 30-60 seconds | all kicked | preserved (loaded from save) |

For an active dev loop, this turns "edit → wait 60s for restart → test" into "edit → drop flag → test" with a roughly 30-60x speedup.

## Patterns that go well with this

### Pattern 1: bot-delivered hotfixes

Bot watches your code repo, pulls new `main.lua` to the server's mod directory, drops the flag. Mod reloads, runs the new code. Production patching without operator intervention.

### Pattern 2: config-only reloads

If your mod reads a config JSON at boot, you can edit the JSON, drop the flag, and the next run picks up the new values. Useful for tuning spawn rates, payouts, cooldowns, etc. without code changes.

### Pattern 3: persistent state across reloads

Write tracking tables to disk on update, read on boot. Survives reloads automatically.

```lua
-- On every state change:
local function persistState()
    local f = io.open(MOD_SAVED_DIR .. "/state.json", "w")
    f:write(jsonEncode(myTrackingTable))
    f:close()
end

-- At boot (runs on every reload):
local function loadState()
    local f = io.open(MOD_SAVED_DIR .. "/state.json", "r")
    if f == nil then return {} end
    local body = f:read("*all"); f:close()
    return jsonDecode(body)
end

local myTrackingTable = loadState()
```

## What this is NOT

To stay honest with anyone you sell or share this with:

- Not "live code patching." Running functions don't get swapped; the whole mod's Lua state is recreated.
- Not a substitute for full server restart in all cases. Some changes (especially hook signature changes) still need a full restart for clean state.
- Not a UE4SS-specific magic feature. The mechanism is just file polling + an API call. Other UE4SS games that expose `RestartCurrentMod` can use the same pattern.
- Not guaranteed crash-free. If your mod has bugs that fire at boot, reload re-triggers them. Test before pushing reload flags in production.

## Reference implementation

The proven-working pattern is in SkinMod's `main.lua` on EVRIMA. The relevant functions are `consumeFlag` (helper) and the body of the polling loop that calls `RestartCurrentMod()`. Total contribution to mod size: ~15 lines.

Verified working as of 2026-05-22 on The Isle EVRIMA dedicated server, UE4SS v3.0.1 Beta.
