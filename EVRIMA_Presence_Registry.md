# EVRIMA presence registry pattern

This is the fix for the broken bulk-enumeration of connected players on The Isle EVRIMA dedicated server. If you're writing any kind of UE4SS Lua mod that needs to iterate "everyone who is currently online," you need this pattern, because every obvious approach is broken in a different way.

## The problem

On EVRIMA, there is no working bulk-enumeration of online players via UE4SS Lua.

`gm.AllPlayerControllers` returns an empty array even with players connected. Silent failure. Your iteration loop runs zero times, you get no error, and you spend half a day wondering why your mod looks like it's working but nothing fires.

`gm.AllPlayerCharacters` returns stale ghost pawns. Touching any method on a ghost pawn (including `GetController()` itself) crashes the server with a native AV.

`FindAllOf("TIPlayerController")` returns stale post-disconnect controllers for a window of a few minutes after a player leaves. Calling methods on them crashes too.

`FindAllOf("TIDinosaurBase")` and similar character-class enumerations are similarly broken.

But `gm:GetControllerBySteamId(steam)` does work. For any steam ID known to be online, it returns a valid controller. The challenge is knowing which steams are online without enumerating.

## The solution: heartbeat hook plus active refresh tick

`/Script/TheIsle.TIPlayerController:SetAdminCred(bool bIsAdmin)` is the heartbeat seed. The function:

- Fires once per controller immediately after connect.
- Fires periodically in steady state. The cadence is bursts of two fires roughly 14 seconds apart, with bursts spaced about 7 to 8 minutes apart in steady state.
- `self` is the controller, no parameter-wrapper staleness concerns.
- The bool parameter reflects engine-side admin status, not your mod's admin list. The function fires unconditionally for both admins and non-admins (bool false for non-admins). Verified by stripping the admin entry from `Game.ini` `[/Script/TheIsle.TIGameStateBase] AdminsSteamIDs=...`, restarting the server, and watching SetAdminCred fire with `p1=bool:false` on the same player's reconnect.

The 7-minute cadence is critical to understand. It is far longer than any reasonable TTL. The fix is not just the seed hook. You also need an active refresh tick that runs every 15 seconds, re-validating each registry entry via `gm:GetControllerBySteamId(steam)`. Healthy entries get `lastSeen` bumped, and entries whose controller returns nil get evicted immediately. The TTL becomes a safety net rather than the primary eviction mechanism.

## The cookbook

Drop this into any mod's `main.lua`. About 50 lines including the helper:

```lua
-- ============================================================================
-- ONLINE PLAYER REGISTRY
-- gm.AllPlayerControllers is BROKEN on EVRIMA dedicated servers; it always
-- returns an empty array even with connected players. This module maintains
-- its own registry, fed by SetAdminCred heartbeat plus chat-hook as a
-- secondary heartbeat source. Entries expire after PRESENCE_EXPIRY_SEC of no
-- refresh OR when the steam's controller goes nil. Controllers are re-derived
-- via gm:GetControllerBySteamId at iteration time; never cached from hook
-- params.
-- ============================================================================

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

-- CRITICAL: active refresh tick. SetAdminCred fires only every 7-8 minutes
-- in steady state, far longer than any reasonable TTL. Without this tick the
-- registry empties between heartbeat bursts and the mod goes dark for several
-- minutes at a time. The 15-second interval re-validates each entry via
-- GetControllerBySteamId; the heartbeat hook just seeds the registry on
-- initial connect.

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

-- Returns the controller's live pawn, or nil if the pawn isn't actually
-- spawned. UE4SS's K2_GetPawn returns a non-nil wrapper around a nullptr
-- pawn when the controller exists but the player hasn't spawned a dino yet
-- (e.g. on spawn-zone select after reconnect). Calling methods on that
-- wrapper crashes the server with a native AV. MUST filter via GetAddress.

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

-- Returns a list of {controller, pawn, steam} for online players.
-- Inline-evicts stale entries (expired or controller went nil).
-- pawn may be nil in the returned tuple if the controller is online but the
-- player is on the spawn-zone-select screen with no dino yet. Callers using
-- pawn must nil-check.

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

Boot-time wiring, at the same point where you call your chat-hook registration:

```lua
registerChatHook()
presenceRegisterHook()
presenceStartRefreshTick()  -- REQUIRED; without this the registry empties
```

## Secondary heartbeat from chat hook

The chat hook can serve as a secondary heartbeat source. Move the presence update BEFORE any chat-enabled gate so it works even on chat-disabled servers:

```lua
RegisterHook("/Script/TheIsle.TIPlayerController:GetChatMessage", function(self, newText, senderCtrlParam, ...)
    -- Presence refresh runs BEFORE any chatEnabled gate
    local s = unwrapParam(senderCtrlParam)
    if s ~= nil then
        local steam = getControllerSteamId(s)
        if steam ~= nil and steam ~= "" then presenceUpdate(steam) end
    end
    if not config.chatEnabled then return end
    -- ...rest of chat handler...
end)
```

## Why each design choice

The pattern looks heavier than it needs to be at first. The reason for each piece:

**Per-mod, not shared.** UE4SS Lua states are isolated between mods. A `_G.<ModAPI>` from one mod is not visible to another. File-based shared registry adds I/O latency plus race conditions on a busy server. Duplicating roughly 50 lines per mod is fine.

**Re-derive the controller per iteration.** Hook parameter wrappers are unstable across ticks. `gm:GetControllerBySteamId(steam)` on each iteration is cheap and always returns a fresh valid pointer for online steams.

**180-second expiry.** Two missed heartbeats before TTL eviction. For UI, this means dead players take roughly 3 minutes to disappear in the worst case. That tradeoff is acceptable because precise disconnect detection doesn't exist on this build (see next section).

**Inline eviction on controller-gone.** If `gm:GetControllerBySteamId(steam)` returns nil, the player is truly gone. Evict immediately instead of waiting the 180-second expiry. Cuts eviction latency in practice.

## Disconnect signals: none usable

Across roughly a week of testing every plausible disconnect hook, exactly zero UFunctions fire on completed disconnects.

Safe logout (lay down and hold H for 60 seconds): only `PrepareSafeLogout` fires (twice per initiation). There is no `SafeLogout` UFunction call on completion.

Menu logout: nothing fires.

Alt+F4: nothing fires.

The server console logs `LogTheIsleJoinData: [...] Left The Server while not being safelogged, ...` lines, but that is an internal C++ log statement, not a Lua-hookable UFunction. The only viable eviction is the heartbeat-expiry pattern documented above.

## Verified UFunctions on the current build

These hook successfully via `RegisterHook` and actually fire during normal play:

- `/Script/TheIsle.TIPlayerController:SetAdminCred(bool)` is the universal heartbeat (about every 90 seconds in churn-heavy testing, 7 minutes in steady state).
- `/Script/TheIsle.TIPlayerController:SetDevCred(bool)` fires alongside SetAdminCred. Could also serve as heartbeat.
- `/Script/TheIsle.TIPlayerController:PrepareSafeLogout()` fires when safe logout starts. Fires twice per initiation.
- `/Script/TheIsle.TIPlayerController:GetChatMessage(...)` fires when chat is received. One fire per receiver in range, hence the dedup requirement on chat handlers.

These hook successfully but never actually fire in normal play (they exist as declared UFunctions but no game code path triggers them):

- `OnPlayerLoginStatusChange`
- `EndLoadingScreen`
- `OpenSpawnZoneSelect`
- `OpenFactionSelect`
- `Logout`
- `SafeLogout`
- `CancelSafeLogout`
- `ClientLogout`
- `ClientKicked`
- `RequestInitialSpawn`
- `SpawnInRandomNest`
- `RequestSpawnInNestAsSpecies`
- `RequestSpawnInNestByKey`
- `UpdatePlayerCredential`
- `PrintLogout`
- `OnPlayerRespawned`
- `OnPawnKicked`
- `RequestSpawnInNest`
- `OnPawnDeath`

These fail to register at all (UFunction not found on this build):

- `K2_PostLogin`, `PostLogin`, `K2_OnLogout`, `HandleStartingNewPlayer`, `GenericPlayerInitialization`, `OnPostLogin` on every game mode candidate
- `ReceivedPlayer`, `K2_OnDestroyed`, `ServerInitConnection` on every player controller candidate

The lesson: do not assume a UFunction by name. Build a spy mod that hooks every plausible candidate first, run normal play, observe which ones fire. Many of these are dead code paths from the game's perspective.

## Closing notes

This is the single highest-leverage pattern in the EVRIMA Lua modding toolkit. Implement it once per mod, use it for every "what players are online" query, and never touch `gm.AllPlayerControllers` again.

A common early mistake is implementing only the heartbeat hook without the 15-second refresh tick. The mod works for the first connection burst then goes dark for 4 minutes between heartbeat bursts. The refresh tick is what makes the pattern usable in production.
