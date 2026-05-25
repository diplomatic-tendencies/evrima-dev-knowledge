# EVRIMA UE4SS Lua helpers reference

These helpers are the building blocks every production mod ends up needing. Most are short (5 to 15 lines). All are exception-guarded for the safety rules described in `EVRIMA_Lua_Safety_Rules.md`.

## findGameMode

Looks up the server's game mode object. The class name differs depending on which gameplay mode the server is running, so the helper tries each candidate in order.

```lua
local function findGameMode()
    local gm = FindFirstOf("BP_SurvivalGameMode_C")
    if gm == nil then gm = FindFirstOf("TISurvivalGameMode") end
    if gm == nil then gm = FindFirstOf("TIGameModeBase") end
    return gm
end
```

`BP_SurvivalGameMode_C` is the BP wrapper for the survival mode. `TISurvivalGameMode` is the native class. `TIGameModeBase` is the parent class that exists regardless of mode. Trying them in order covers every server configuration.

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

Safely reads the steam ID string from a controller. Returns nil if the controller is nil or if any step in the chain fails.

```lua
local function getControllerSteamId(ctrl)
    if ctrl == nil then return nil end
    local sId
    pcall(function() sId = ctrl:GetSteamId() end)
    if sId == nil then return nil end
    local s
    pcall(function() s = sId:ToString() end)
    return s
end
```

The pcall wraps are defensive against edge cases (controller has no associated steam, the FName-to-string conversion fails on stale wrappers).

## log

Simple logger that prefixes with the mod name and goes to `UE4SS.log` via `print()`. Customize the prefix per mod for grep-friendly output.

```lua
local function log(msg)
    print("[YourModName] " .. tostring(msg))
end
```

`print()` in UE4SS Lua writes to `UE4SS.log`. The `[YourModName]` prefix is the convention every mod in the bundle uses; it makes per-mod filtering trivial when you have multiple mods logging to the same file.

For higher-volume logging, add log levels and a per-level toggle:

```lua
local LOG_VERBOSE = true

local function log(msg) print("[YourModName] " .. tostring(msg)) end
local function vlog(msg) if LOG_VERBOSE then log(msg) end end
```

## safeNotify

Sends a chat notification to a specific player. Safe to call from a poll or timer tick; NOT safe from inside a hook callback (safety rule 5).

```lua
local function safeNotify(steamId, message)
    local gm = findGameMode()
    if gm == nil then return false, "no-gameMode" end

    local controller
    pcall(function() controller = gm:GetControllerBySteamId(steamId) end)
    if controller == nil then return false, "no-controller" end

    local text = message
    if FText ~= nil then
        local ok, ft = pcall(function() return FText(message) end)
        if ok and ft ~= nil then text = ft end
    end

    local okN, errN = pcall(function() controller:ClientShowNotification(text) end)
    return okN, errN
end
```

The fresh `findGameMode()` plus `gm:GetControllerBySteamId(steamId)` chain produces a valid pointer with the right lifetime. A stored controller from a hook param does not.

The `FText` wrap is required because `ClientShowNotification` expects FText, not a raw string. If `FText` isn't exposed in this UE4SS build, the helper falls back to passing the raw string (which usually works but is technically incorrect).

## notify (convenience wrapper)

A shorter name for `safeNotify`, often used in chat-command handlers:

```lua
local function notify(steamId, msg)
    return safeNotify(steamId, msg)
end
```

Pure convenience; the underlying behavior is `safeNotify`.

## bumpPureBlack

Nudges any pure-zero RGB channel to 0.01 to defeat the engine's pure-black sentinel. Required for `FCustomizerDataBase` color writes because the engine treats `FLinearColor(0, 0, 0)` as "unset" in some replication paths.

```lua
local function bumpPureBlack(c)
    if c == nil then return nil end
    return {
        R = (c.R == 0.0) and 0.01 or c.R,
        G = (c.G == 0.0) and 0.01 or c.G,
        B = (c.B == 0.0) and 0.01 or c.B,
        A = c.A or 1.0,
    }
end
```

The visual difference between 0.0 and 0.01 is imperceptible in-game; the practical difference is colors persisting across login.

## isAdmin / isAdminOrAllowed

Application-layer permission check. The exact implementation depends on your admin model.

For a simple steam-list check:

```lua
local ADMIN_STEAMS = {
    ["76561198XXXXXXX"] = true,
    ["76561198YYYYYYY"] = true,
}

local function isAdmin(steam)
    if steam == nil then return false end
    return ADMIN_STEAMS[tostring(steam)] == true
end
```

For a config-file-driven check, load the list from `Mods/YourMod/Saved/admins.txt` at boot. For a role-based check, integrate with your bot's database.

The `isAdminOrAllowed` variant in the BodyDrop architecture doc is a wider permission check that includes both admins and an explicitly-allowed list (for trusted moderators who shouldn't have full admin access).

## encodeJson / decodeJson

JSON serialization. UE4SS Lua doesn't include a JSON library by default, so most mods bundle a small pure-Lua JSON encoder/decoder.

The standard choice is `rxi/json.lua` or similar small open-source library (about 350 lines, MIT licensed). Drop the file into `Mods/YourMod/Scripts/json.lua` and require it:

```lua
local json = require("json")

local function encodeJson(obj)
    return json.encode(obj)
end

local function decodeJson(s)
    local ok, result = pcall(json.decode, s)
    if not ok then return nil end
    return result
end
```

The `decodeJson` wrapper catches parse errors and returns nil instead of throwing. Important for tail-reading NDJSON files where one bad line shouldn't break the whole loop.

## appendLines

Batched file append. The naive open-write-close per line crushes disk performance under load; batching the lines is the fix.

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

Used by the PlayerStats snapshot loop and the CommandBridge results writer. Call once per batch, not once per line.

## stripBlueprintPrefix

Strips the `"BlueprintGeneratedClass "` prefix that `GetClass():GetFullName()` returns. Useful for normalizing class paths for storage or comparison.

```lua
local function stripBlueprintPrefix(s)
    if s == nil then return nil end
    s = tostring(s)
    local prefix = "BlueprintGeneratedClass "
    if s:sub(1, #prefix) == prefix then
        return s:sub(#prefix + 1)
    end
    return s
end
```

Some UE versions return `"BlueprintGeneratedClass /Game/..."`; others return just `/Game/...`. The helper handles both.

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

A wrapper around `RegisterHook` that catches registration failures and logs them, plus optionally retries on the next tick if the UFunction isn't found yet.

```lua
local function registerSafeHook(fnName, label, callback)
    local ok, err = pcall(function()
        RegisterHook(fnName, callback)
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

## loadSavedColors (example per-player file loader)

Pattern for loading per-steam JSON files:

```lua
local function loadSavedColors(steam)
    if steam == nil or steam == "" then return nil end
    local path = "Mods/SkinMod/Saved/skins/" .. tostring(steam) .. ".json"
    local f = io.open(path, "r")
    if f == nil then return nil end
    local body = f:read("*a")
    f:close()
    if body == nil or body == "" then return nil end
    local obj = decodeJson(body)
    if obj == nil then return nil end
    return obj.colors
end
```

Adapt the path and the expected schema to your mod. The pattern is the same for any per-player file: build the path from steam, open-read-close, decode, return the relevant field.

## writeResult (CommandBridge helper)

Append a result line to the shared results.ndjson file:

```lua
local function writeResult(request, result)
    local line = {
        id = request.id,
        ts = os.time(),
        status = result.status or "ok",
        cmd = request.cmd,
        steam = request.steam,
        detail = result.detail or "",
    }
    local encoded = encodeJson(line)
    if encoded == nil then return false end
    return appendLines("Mods/CommandBridge/Saved/results.ndjson", { encoded })
end
```

The request's `id` field is echoed back so the bot can correlate request to response.

## pollInbox (CommandBridge helper)

Tail an inbox NDJSON file, dispatch each new line to a handler:

```lua
local inboxPositions = {}

local function pollInbox(subMod, filePath, handler)
    local f = io.open(filePath, "r")
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
```

Run from a recurring tick:

```lua
LoopInGameThreadWithDelay(500, function()
    for subMod, h in pairs(handlers) do
        local filePath = "Mods/CommandBridge/Saved/inbox/" .. subMod .. ".ndjson"
        pollInbox(subMod, filePath, h)
    end
end)
```

## Boot wiring template

A typical mod's boot sequence pulls in these helpers and wires the standard infrastructure. The template:

```lua
-- 1. Helpers (the contents of this document)
-- 2. Mod-specific state and config
local config = loadConfig()

-- 3. Chat hook (your own, depending on what commands you handle)
registerChatHook()

-- 4. Presence registry (every mod that touches online players)
presenceRegisterHook()
presenceStartRefreshTick()

-- 5. Mod-specific recurring ticks
LoopInGameThreadWithDelay(5000, snapshotOnce)        -- if you're emitting events
LoopInGameThreadWithDelay(500, pollInboxForCommands) -- if you handle bot commands

-- 6. One-shot startup tasks
LoopInGameThreadWithDelay(3000, runStartupTasks)

log("YourModName loaded; version=" .. MOD_VERSION)
```

The version log line is the boot-complete signal that watcher scripts (and the server restart loop) typically grep for.

## Closing notes

These helpers are the underlying infrastructure every mod uses. Drop them into a `Mods/YourMod/Scripts/helpers.lua` file and `require` it from `main.lua`, or paste them inline if you prefer fewer files. Either approach works; the per-mod-isolation rule means there's no benefit to a shared helpers module across mods.

Most of the helpers are exception-guarded against the specific failure modes documented in `EVRIMA_Lua_Safety_Rules.md`. Stripping the pcall wraps to make the code shorter usually causes a real crash a few weeks later when an edge case appears. Keep the wraps.
