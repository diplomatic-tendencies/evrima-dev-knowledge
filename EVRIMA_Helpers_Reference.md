# EVRIMA UE4SS Lua helpers reference

These helpers are the building blocks every production mod ends up needing. Most are short (5 to 15 lines). All are exception-guarded for the safety rules described in `EVRIMA_Lua_Safety_Rules.md`.

## findGameMode

Looks up the server's game mode object. The class name differs depending on which gameplay mode the server is running, so the helper tries each candidate in order. Each `FindFirstOf` is pcall-wrapped because that call can crash for a class with no live instances (safety rule 3a).

```lua
local function findGameMode()
    local candidates = { "BP_SurvivalGameMode_C", "TISurvivalGameMode", "TIGameModeBase", "GameModeBase" }
    for _, name in ipairs(candidates) do
        local gm
        pcall(function() gm = FindFirstOf(name) end)
        if gm ~= nil then return gm end
    end
    return nil
end
```

`BP_SurvivalGameMode_C` is the BP wrapper for the survival mode. `TISurvivalGameMode` is the native class. `TIGameModeBase` is the parent class that exists regardless of mode. `GameModeBase` is the engine base fallback. Trying them in order covers every server configuration.

Don't cache the returned pointer across ticks (same stale-wrapper issue as controller wrappers). Call `findGameMode()` fresh each time you need it. The call is cheap; `FindFirstOf` on a known-loaded class is roughly a hash lookup.

Returns nil only if the game hasn't loaded a map yet, which shouldn't happen post-boot.

## livePawnFromCtrl

Returns a controller's live pawn, or nil if the pawn isn't actually spawned. Safety rule 9a in the safety doc covers why this is necessary: `K2_GetPawn()` returns a non-nil wrapper around a null pawn when the player exists but hasn't spawned a dino yet (spawn-zone select after reconnect, mid-respawn between death and species pick, etc.). Calling methods on that wrapper crashes the server.

```lua
local function livePawnFromCtrl(ctrl)
    if ctrl == nil then return nil end
    local pawn
    pcall(function() pawn = ctrl:K2_GetPawn() end)
    if pawn == nil then return nil end
    local addr
    pcall(function() addr = pawn:GetAddress() end)
    if addr == nil or addr == 0 then return nil end
    return pawn
end
```

The `GetAddress() ~= 0` check is what distinguishes a real pawn from a nullptr wrapper. Use this helper everywhere you derive a pawn from a controller via an indirect lookup.

## unwrapParam

Extracts the underlying UObject from a `RegisterHook` callback parameter wrapper. The wrapper is what UE4SS passes to hook callbacks; the underlying UObject is what your hook logic actually wants.

```lua
local function unwrapParam(p)
    if p == nil then return nil end
    local obj
    pcall(function() obj = p:get() end)
    return obj
end
```

Used inside hook callbacks to convert `param` to a usable `UObject`. Don't cache the returned object across ticks; the underlying memory may be reused (safety rule 6).

## getControllerSteamId

Safely reads the steam ID string from a controller. Returns `""` (empty string) if any step fails — production callers check `steam ~= "" and steam ~= nil` rather than just nil.

```lua
local function safeString(value)
    if value == nil then return "" end
    -- Try native tostring first
    local ok, s = pcall(function() return tostring(value) end)
    if ok and type(s) == "string" and s ~= "" and not s:find("^UObject") then return s end
    -- Try :ToString() method (for FString/FName userdata)
    local okT, t = pcall(function() return value:ToString() end)
    if okT and type(t) == "string" then return t end
    return ""
end

local function getControllerSteamId(ctrl)
    if ctrl == nil then return "" end
    -- Primary path: GetSteamId() UFunction
    local sId
    pcall(function() sId = ctrl:GetSteamId() end)
    if sId ~= nil then
        local s = safeString(sId)
        if s ~= "" and not s:find("^UObject") then return s end
    end
    -- Fallback path: SteamId field-access (works on wrappers where GetSteamId returns garbage)
    local field
    pcall(function() field = ctrl.SteamId end)
    if field ~= nil then
        local s = safeString(field)
        if s ~= "" then return s end
    end
    return ""
end
```

The pcall wraps are defensive against edge cases (controller has no associated steam, the FName-to-string conversion fails on stale wrappers). The `SteamId` field fallback is necessary because `GetSteamId()` sometimes returns wrappers that stringify as `"UObject: 0x..."` garbage; the underlying property field is more stable on those cases.

## log

Simple logger that prefixes with the mod name and goes to `UE4SS.log` via `print()`. The trailing `\n` is required — UE4SS's `print()` does NOT append a newline; without it, log lines from a single mod run together into one giant line.

```lua
local MOD_NAME = "YourModName"

local function log(msg)
    print(string.format("[%s] %s\n", MOD_NAME, tostring(msg)))
end
```

`print()` in UE4SS Lua writes to `UE4SS.log`. The `[YourModName]` prefix is the convention every production mod uses; it makes per-mod filtering trivial when you have multiple mods logging to the same file.

For higher-volume logging, add log levels and a per-level toggle:

```lua
local LOG_VERBOSE = true

local function log(msg) print(string.format("[%s] %s\n", MOD_NAME, tostring(msg))) end
local function vlog(msg) if LOG_VERBOSE then log(msg) end end
```

## makeText

Wraps a Lua string as `FText` for UFunctions that require it (notably `ClientShowNotification`). Falls back to the raw string if UE4SS doesn't expose `FText`.

```lua
local function makeText(message)
    if FText == nil then return message end
    local ok, ft = pcall(function() return FText(message) end)
    if ok and ft ~= nil then return ft end
    return message
end
```

## safeNotify

Sends a chat notification to a specific player. Safe to call from a poll or timer tick; NOT safe from inside a hook callback (safety rule 5).

Production mods differ in return shape. The pattern below uses `(ok, reason)` (matching DinoStorage). Other mods use single-bool or no return — pick whichever shape your caller code expects and stay consistent within a mod.

```lua
local function safeNotify(steamId, message)
    if steamId == nil or steamId == "" then return false, "no-steam" end
    if message == nil or message == "" then return false, "no-message" end
    local gm = findGameMode()
    if gm == nil then return false, "no-gameMode" end

    local controller
    pcall(function() controller = gm:GetControllerBySteamId(steamId) end)
    if controller == nil then return false, "no-controller" end

    local text = makeText(message)
    local okN, errN = pcall(function() controller:ClientShowNotification(text) end)
    if not okN then return false, "ClientShowNotification failed: " .. tostring(errN) end
    return true, "ok"
end
```

The fresh `findGameMode()` plus `gm:GetControllerBySteamId(steamId)` chain produces a valid pointer with the right lifetime. A stored controller from a hook param does not. Note: `ClientShowNotification` is intermittently flaky in production — sometimes returns ok=true without the popup reaching the client, especially right after a player spawns. Defer + retry or hold off notifying within a few seconds of fresh connect.

## queueNotify (production pattern — defer and drain)

Because notify shouldn't be called from inside hooks (safety rule 5), production mods queue messages in the hook and drain from a tick. Pattern:

```lua
local pendingNotifies = {}

local function queueNotify(steamId, message)
    if steamId == nil or steamId == "" or message == nil or message == "" then return end
    pendingNotifies[#pendingNotifies + 1] = { steam = steamId, msg = message }
end

-- In your recurring poll tick:
LoopInGameThreadWithDelay(3000, function()
    if #pendingNotifies == 0 then return end
    local drain = pendingNotifies
    pendingNotifies = {}
    for _, n in ipairs(drain) do
        pcall(function() safeNotify(n.steam, n.msg) end)
    end
end)
```

This is the safe-from-hook pattern. Direct `safeNotify` from a chat-hook callback violates rule 5; queueing + draining from a tick fixes it.

## bumpPureBlack (inline guard, not a standalone helper)

The engine appears to treat `FLinearColor(0, 0, 0)` as "unset" in some replication paths. Production SkinMod guards against this with an inline check inside its apply path — the bump triggers only when **all three** RGB channels are zero, not per-channel:

```lua
-- Inside applySkin, right before writing each color field:
if r == 0 and g == 0 and b == 0 then r, g, b = 0.01, 0.01, 0.01 end
cdata[fieldName].R = r
cdata[fieldName].G = g
cdata[fieldName].B = b
cdata[fieldName].A = 1.0
```

The visual difference between 0.0 and 0.01 is imperceptible in-game; the practical difference is colors persisting cleanly. Note: this is an inline guard in `applySkin`, not a standalone reusable function in the live codebase. The "sentinel bug" itself is folklore — no probe has cleanly reproduced the relog-reversion the workaround addresses — but the bump is cheap and the production code keeps it.

## isAdmin

Application-layer permission check. The exact implementation depends on your admin model. Two production patterns:

**Config-driven** (BodyDrop, PlayerStats — recommended):

```lua
-- After loadConfig() populates config.adminSteamIds = {"76561198XXX", ...}
local function isAdmin(steam)
    if steam == nil then return false end
    for _, s in ipairs(config.adminSteamIds or {}) do
        if tostring(s) == tostring(steam) then return true end
    end
    return false
end
```

**Static table** (DinoStorage):

```lua
ADMIN_STEAM_IDS = {       -- intentionally global so reload.flag preserves it
    ["76561198XXXXXXX"] = true,
}

local function isAdmin(steam)
    return steam ~= nil and ADMIN_STEAM_IDS[tostring(steam)] == true
end
```

For a role-based check (Discord roles, etc.), integrate with your bot's database via CommandBridge instead.

## JSON read helpers (no `require`, no external library)

UE4SS Lua doesn't include a JSON library, and production mods do NOT pull one in via `require`. Instead they use ad-hoc regex readers for the few fields they actually need. Pattern:

```lua
local function jsonReadString(body, fieldName)
    return string.match(body or "", '"' .. fieldName .. '"%s*:%s*"([^"]*)"')
end

local function jsonReadNumber(body, fieldName)
    return tonumber(string.match(body or "", '"' .. fieldName .. '"%s*:%s*(-?%d+%.?%d*)'))
end

local function jsonReadBool(body, fieldName)
    local v = string.match(body or "", '"' .. fieldName .. '"%s*:%s*([%a]+)')
    if v == "true"  then return true end
    if v == "false" then return false end
    return nil
end
```

For writes, production mods build NDJSON via `string.format` and concatenation (no recursive encoder needed for flat-ish shapes). PlayerStats and CommandBridge include a small recursive `jsonValue` builder for nested-object events — copy from those mods if you need it.

Why no `require("json")`: keeps the mod dependency-free, avoids `package.path` issues across UE4SS builds, and the read patterns above cover everything production needs. If your mod has genuinely complex parsing needs, drop in `rxi/json.lua` (small, MIT-licensed, ~350 lines) under your own `Mods/<YourMod>/Scripts/`, but be aware no live mod does this.

## jsonEscape (write-side helper)

For escaping strings going INTO an NDJSON write:

```lua
local function jsonEscape(s)
    if s == nil then return "" end
    s = tostring(s)
    s = s:gsub("\\", "\\\\")
    s = s:gsub('"', '\\"')
    s = s:gsub("\n", "\\n")
    s = s:gsub("\r", "\\r")
    s = s:gsub("\t", "\\t")
    return s
end
```

Use when building lines like `string.format('{"msg":"%s"}', jsonEscape(text))`.

## appendLine (singular)

Append a single line to a file:

```lua
local function appendLine(path, line)
    local f = io.open(path, "ab")
    if f == nil then return false end
    f:write(line); f:write("\n"); f:close()
    return true
end
```

Production mods call this once per line. There's no batched-append helper in the live codebase — file IO is fast enough for the per-tick volumes involved (1 PlayerStats line per player per 5s; a few command results per second peak). If you do need batching, write it yourself by opening the file once and writing N lines before closing.

## stripClassPrefix

`GetClass():GetFullName()` returns strings like `"BlueprintGeneratedClass /Game/.../BP_Tyrannosaurus_C"` or `"TIDinosaurBase /Game/..."`. Strips the leading-word-plus-whitespace prefix to get just the path.

```lua
local function stripClassPrefix(s)
    if s == nil then return nil end
    s = tostring(s)
    return string.match(s, "^%S+%s+(.+)$") or s
end
```

The regex pattern matches `<non-whitespace><whitespace><rest>`. Falls through to the original string if no prefix was present. Production mods use this naming convention (`stripObjectTypePrefix` in PlayerStats, anonymous helper in DinoStorage). It correctly handles all the class-name prefixes the engine emits, not just `BlueprintGeneratedClass`.

## Presence registry helpers

Five helpers that together form the presence registry pattern. The full discussion is in `EVRIMA_Presence_Registry.md`; the implementations are reproduced here for completeness.

```lua
local presenceRegistry = {}
local PRESENCE_EXPIRY_SEC = 180

local function presenceUpdate(steam)
    if steam == nil or steam == "" then return end
    local s = tostring(steam)
    if not presenceRegistry[s] then
        presenceRegistry[s] = { firstSeen = os.time() }
    end
    presenceRegistry[s].lastSeen = os.time()
end

local function presenceRegisterHook()
    local ok, err = pcall(function()
        RegisterHook("/Script/TheIsle.TIPlayerController:SetAdminCred", function(ctrlParam, _bool)
            local self_
            pcall(function() self_ = ctrlParam:get() end)
            if self_ == nil then return end
            local sId
            pcall(function() sId = self_:GetSteamId() end)
            if sId == nil then return end
            local steamStr
            pcall(function() steamStr = sId:ToString() end)
            if steamStr ~= nil and tostring(steamStr) ~= "" then
                presenceUpdate(steamStr)
            end
        end)
    end)
    if ok then log("Presence heartbeat hook registered (SetAdminCred)")
    else log("Presence heartbeat hook FAILED: " .. tostring(err)) end
end

local function presenceStartRefreshTick()
    if LoopInGameThreadWithDelay == nil then return end
    LoopInGameThreadWithDelay(15000, function()
        local gm = findGameMode()
        if gm == nil then return end
        local now = os.time()
        for steam, entry in pairs(presenceRegistry) do
            local ctrl
            pcall(function() ctrl = gm:GetControllerBySteamId(steam) end)
            if ctrl == nil then
                presenceRegistry[steam] = nil
            else
                entry.lastSeen = now
            end
        end
    end)
    log("Presence refresh tick started (15s interval)")
end

local function enumerateOnlinePlayers()
    local results = {}
    local gm = findGameMode()
    if gm == nil then return results end
    local now = os.time()
    for steam, entry in pairs(presenceRegistry) do
        if (now - entry.lastSeen) > PRESENCE_EXPIRY_SEC then
            presenceRegistry[steam] = nil
        else
            local ctrl
            pcall(function() ctrl = gm:GetControllerBySteamId(steam) end)
            if ctrl == nil then
                presenceRegistry[steam] = nil
            else
                local pawn = livePawnFromCtrl(ctrl)
                results[#results + 1] = { controller = ctrl, pawn = pawn, steam = steam }
            end
        end
    end
    return results
end
```

### Wiring (required, easy to miss)

These helpers must be **called** at boot, not just defined. Pasting the function bodies into your module without invoking `presenceRegisterHook()` and `presenceStartRefreshTick()` leaves the registry empty forever. The symptom is `enumerateOnlinePlayers()` always returning `{}` even with players connected.

```lua
registerChatHook()           -- your own chat hook setup
presenceRegisterHook()       -- installs the SetAdminCred heartbeat hook
presenceStartRefreshTick()   -- starts the 15-second registry refresh tick
```

`presenceRegisterHook` is what feeds the registry from connect plus heartbeat events. `presenceStartRefreshTick` is what keeps entries alive between SetAdminCred bursts (which fire only every 7 to 8 minutes in steady state). If either is missing, the registry stays empty or empties out within a few minutes of every burst.

After boot, verify by checking `UE4SS.log` for `Presence heartbeat hook registered (SetAdminCred)` and `Presence refresh tick started (15s interval)`. If those lines are missing, the wiring did not run.

## registerSafeHook

A wrapper around `RegisterHook` that catches registration failures and logs them. Production version (DinoStorage) **double-pcalls**: once around the registration itself, once around the user callback. The second pcall makes hook callbacks crash-resistant — if your callback errors out, it doesn't propagate up the hook chain.

```lua
local function registerSafeHook(fnName, label, callback)
    local ok, err = pcall(function()
        RegisterHook(fnName, function(...)
            -- Wrap user callback in its own pcall so a single bad fire
            -- doesn't bring down the hook or any subsequent fires.
            pcall(callback, ...)
        end)
    end)
    if ok then
        log("Hook registered: " .. label .. " (" .. fnName .. ")")
    else
        log("Hook FAILED: " .. label .. " (" .. fnName .. "): " .. tostring(err))
    end
    return ok
end
```

For UFunctions that may not be loaded yet at mod-boot time, defer the registration to a tick after the engine has fully initialized:

```lua
LoopInGameThreadWithDelay(5000, function()
    registerSafeHook("/Script/TheIsle.SomeLateLoadedClass:SomeFunc", "label", callback)
end)
```

## loadSkin (example per-player file loader)

Pattern for loading per-steam JSON files. The SkinMod schema is flat: top-level keys are the engine field names (`BodyColor`, `MarkingsColor`, etc.) with `{r,g,b,a}` lowercase sub-keys. No `colors` envelope, no `steam`/`updatedAt` metadata.

```lua
local function loadSkin(steam)
    if steam == nil or steam == "" then return nil end
    local path = "Mods/SkinMod/Saved/skins/" .. tostring(steam) .. ".json"
    local f = io.open(path, "r")
    if f == nil then return nil end
    local body = f:read("*a")
    f:close()
    if body == nil or body == "" then return nil end

    -- Parse: matches each top-level "<FieldName>":{...} block
    local out = {}
    for fieldName, colorBlock in string.gmatch(body, '"([%w_]+)"%s*:%s*({[^}]*})') do
        local r = tonumber(string.match(colorBlock, '"r"%s*:%s*(-?%d+%.?%d*)'))
        local g = tonumber(string.match(colorBlock, '"g"%s*:%s*(-?%d+%.?%d*)'))
        local b = tonumber(string.match(colorBlock, '"b"%s*:%s*(-?%d+%.?%d*)'))
        local a = tonumber(string.match(colorBlock, '"a"%s*:%s*(-?%d+%.?%d*)')) or 1.0
        if r and g and b then
            out[fieldName] = { r = r, g = g, b = b, a = a }
        end
    end
    return out
end
```

Returns a table keyed by engine field name (`out.BodyColor.r`, etc.) — no `.colors` wrapping. Adapt the path and field names to your mod's schema.

## CommandBridge inbox + results format (for sub-mod implementers)

If you're implementing a sub-mod that CommandBridge will route commands to, you need a `pollInbox` loop reading `Mods/<YourMod>/Saved/inbox.ndjson` (inbox lives in YOUR mod's Saved/ dir, not under CommandBridge) and a write to the shared results file.

**Inbox-tail pattern (atomic-rename style, used by BodyDrop and PlayerStats):**

```lua
local INBOX_PATH = "Mods/YourMod/Saved/inbox.ndjson"
local SHARED_RESULTS = "Mods/CommandBridge/Saved/results.ndjson"

local function fileExists(path)
    local f = io.open(path, "rb"); if f == nil then return false end
    f:close(); return true
end

local function pollInbox()
    if not fileExists(INBOX_PATH) then return end
    -- Atomic-ish rename so the bridge can keep writing while we process
    local stash = INBOX_PATH .. ".processing"
    os.remove(stash)
    os.rename(INBOX_PATH, stash)
    local f = io.open(stash, "rb"); if f == nil then return end
    local body = f:read("*a"); f:close()
    if body == nil or body == "" then os.remove(stash); return end

    for line in body:gmatch("[^\n]+") do
        local id    = jsonReadString(line, "id")
        local steam = jsonReadString(line, "steam") or ""
        -- args.tokens shape: ["set","interval","60"]
        local argsBlock = string.match(line, '"args"%s*:%s*%[([^%]]*)%]')
        local tokens = {}
        if argsBlock then
            for s in string.gmatch(argsBlock, '"([^"]*)"') do
                tokens[#tokens + 1] = s
            end
        end

        local ok, msg = pcall(function() return handleCommand(steam, tokens) end)
        if not ok then msg = "error: " .. tostring(msg) end
        appendLine(SHARED_RESULTS, buildResultLine(id, steam, tokens, ok, msg))
    end
    os.remove(stash)
end
```

**Result-line shape** (matches what the live bridge expects):

```lua
local function buildResultLine(id, steam, tokens, ok, msg)
    local tokensJson = "["
    for i, t in ipairs(tokens) do
        if i > 1 then tokensJson = tokensJson .. "," end
        tokensJson = tokensJson .. '"' .. jsonEscape(t) .. '"'
    end
    tokensJson = tokensJson .. "]"
    return string.format(
        '{"id":"%s","ts":%d,"source":"YourMod","steam":"%s","args":%s,"ok":%s,"msg":"%s"}',
        tostring(id or ""), os.time(), tostring(steam), tokensJson,
        tostring(ok == true), jsonEscape(tostring(msg)))
end
```

Field set is `{id, ts, source, steam, args, ok, msg}`. **Note**: the bridge's own queued-ACK lines use `{id, ts, verb, steam, ok, msg}` (no `source`, no `args` array). Bots correlating results by `id` will see both lines for routed verbs; use `source` field's presence/value to distinguish the bridge ACK from your sub-mod's final result.

## safeCall

The dominant idiom for top-level tick functions. Wraps `pcall` and logs label-prefixed failures so a crashing tick doesn't kill the poll loop silently.

```lua
local function safeCall(label, fn)
    local ok, err = pcall(fn)
    if not ok then log(string.format("safeCall(%s) failed: %s", label, tostring(err))) end
    return ok, err
end
```

Use everywhere your tick callback does meaningful work:

```lua
LoopInGameThreadWithDelay(POLL_INTERVAL_MS, function()
    safeCall("pollInbox", pollInbox)
    safeCall("autoRestore", autoRestore)
    safeCall("drainNotifies", drainNotifies)
end)
```

## consumeFlag (read-then-delete trigger pattern)

Used by every mod for `reload.flag` and command-triggered flags. Reads the file body, deletes the file, returns the body trimmed (or nil if absent/empty).

```lua
local function consumeFlag(path)
    local f = io.open(path, "rb"); if f == nil then return nil end
    local body = f:read("*all") or ""; f:close()
    os.remove(path)
    body = body:gsub("^%s+", ""):gsub("%s+$", "")
    if body == "" then return nil end
    return body
end
```

Drop a token into the flag file from outside the server, the poll tick consumes it next iteration.

## writeAll (atomic file write — production pattern)

Two production patterns. Plain write (BodyDrop, PlayerStats, CommandBridge):

```lua
local function writeAll(path, body)
    local f = io.open(path, "wb"); if f == nil then return false end
    f:write(body); f:close(); return true
end
```

Atomic write via temp + rename (DinoStorage, SkinMod) — survives crashes mid-write:

```lua
local function writeAllAtomic(path, body)
    local tmp = path .. ".tmp"
    local f = io.open(tmp, "wb"); if f == nil then return false end
    f:write(body); f:close()
    os.remove(path)
    return os.rename(tmp, path)
end
```

Use the atomic variant for files that must never be half-written (per-player state, index files).

## Boot wiring template

A typical mod's boot sequence pulls in these helpers and wires the standard infrastructure. Real production pattern:

```lua
local MOD_NAME = "YourMod"
local MOD_VERSION = "v001"
local POLL_INTERVAL_MS = 3000  -- 1000ms (CB, PS), 3000ms (SK, DS), 5000ms (BD); never 500ms

local function log(msg) print(string.format("[%s] %s\n", MOD_NAME, tostring(msg))) end

-- ... helper definitions ...

log(string.format("Loading; version=%s", MOD_VERSION))

-- Chat hook + presence registry first (no config dependency)
registerChatHook()
presenceRegisterHook()
presenceStartRefreshTick()

-- One-shot boot tick: load config, fire any startup work, then self-cancel
if LoopInGameThreadWithDelay ~= nil then
    local hbHandle
    hbHandle = LoopInGameThreadWithDelay(5000, function()
        log(string.format("Boot; version=%s", MOD_VERSION))
        safeCall("loadConfig", function() loadConfig("boot") end)
        if hbHandle ~= nil and CancelDelayedAction ~= nil then
            CancelDelayedAction(hbHandle)   -- one-shot: cancel after first fire
        end
    end)

    -- Steady-state poll loop
    LoopInGameThreadWithDelay(POLL_INTERVAL_MS, function()
        safeCall("pollInput", pollInput)
        safeCall("autoRestore", autoRestore)
        safeCall("drainNotifies", drainNotifies)

        local reload = consumeFlag(RELOAD_FLAG)
        if reload ~= nil and RestartCurrentMod ~= nil then
            log(string.format("RELOAD; token=%s", reload))
            RestartCurrentMod()
        end
    end)
end

log(string.format("Loaded; version=%s", MOD_VERSION))
```

Key points production mods get right:
- **`Loading;` log at top, `Loaded;` log at bottom** — paired markers so watchers can detect partial boot.
- **`loadConfig` runs inside a 5000ms one-shot tick, NOT at top level** — gives UE4SS time to fully initialize before file IO.
- **`CancelDelayedAction(hbHandle)` makes the boot tick one-shot** — without this, `LoopInGameThreadWithDelay` keeps re-firing every 5s.
- **Poll interval is 1000ms or 3000ms, never 500ms** — anything faster wastes CPU without meaningful latency improvement.
- **Every tick callback is `safeCall`-wrapped** — one bad fire shouldn't kill the loop.
- **`reload.flag` is consumed from the poll loop**, not top-level.

## Closing notes

These helpers are the underlying infrastructure every mod uses. Drop them into a `Mods/YourMod/Scripts/helpers.lua` file and `require` it from `main.lua`, or paste them inline if you prefer fewer files. Either approach works; the per-mod-isolation rule means there's no benefit to a shared helpers module across mods.

Most of the helpers are exception-guarded against the specific failure modes documented in `EVRIMA_Lua_Safety_Rules.md`. Stripping the pcall wraps to make the code shorter usually causes a real crash a few weeks later when an edge case appears. Keep the wraps.
