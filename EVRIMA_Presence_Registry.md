# EVRIMA presence registry pattern

A self-maintained registry of connected players, fed by a heartbeat hook. It is the pattern I reach for whenever a mod needs to iterate "everyone online," and it stays the production-safe choice even though the engine's own player collections are not as broken as I first reported.

That correction belongs up front, because an earlier version of this document (and `EVRIMA_Lua_Safety_Rules.md` rule 3) claimed there was no working bulk enumeration at all. That was wrong, and the mistake was mine: I read a TSet with array syntax, got nothing back, and concluded it was empty. The real picture is below. The registry still earns its place — its disconnect handling is the part that is actually proven, which is the hard part — but it is no longer the only option, and the docs should not have said it was.

## The enumeration paths

`gm.AllPlayerControllers` is populated, not empty. The catch is that it is a **TSet**, not a TArray. Reading it with `#` or `[i]` returns nothing from a set no matter how many elements it holds, which is exactly the "always empty" symptom I originally chased. Iterate it with `:ForEach(function(elem) ... end)`; a TSet's ForEach passes the element as the first argument, not the `(index, element)` pair a TArray gives you. Verified populated and iterable, solo, on the 2026-06-06 build.

`GameState.PlayerArray` is the cleaner path: a standard `TArray<APlayerState*>`, reached via `gm:GetWorld().GameState` or `FindFirstOf("TIGameStateBase")`. It is array-indexable, and `playerArray[i]:GetOwningController()` followed by `:GetSteamId():ToString()` hands you the controller and steam id directly. Prefer this for new code.

What neither collection is verified to do — and the reason the registry still ships — is behave under disconnects and at multiplayer scale. The 2026-06-06 check was solo. These collections are engine-maintained, so they *should* drop a leaving player cleanly, unlike the `FindAllOf` paths below; but UE keeps a brief `InactivePlayerArray` window after a disconnect, and I have not verified that a recently-departed player doesn't linger in `PlayerArray` long enough to crash a `GetOwningController()` call on a freed object. Until that is tested at two-plus players with a real disconnect, treat the collections as a good online snapshot and the registry as the enumerator you trust in production.

The paths that are genuinely broken, and stay broken:

`gm.AllPlayerCharacters` returns stale ghost pawns. Touching any method on a ghost pawn, `GetController()` included, crashes the server with a native AV.

`FindAllOf("TIPlayerController")` returns stale post-disconnect controllers for a window of a few minutes after a player leaves, and calling methods on them crashes. `FindAllOf("TIDinosaurBase")` and similar character-class enumerations have the same problem.

`gm:GetControllerBySteamId(steam)` works for any online steam and returns a fresh valid controller. The registry's actual job is the narrow one of knowing which steams are online; resolving a known steam to a controller is already solved.

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
-- The engine's player collections (gm.AllPlayerControllers, a TSet; and
-- GameState.PlayerArray) do work for an online snapshot, but their disconnect
-- behavior is unverified at multiplayer scale, so this module maintains its own
-- registry whose eviction IS proven. Fed by SetAdminCred heartbeat plus chat-hook
-- as a secondary source. Entries expire after PRESENCE_EXPIRY_SEC of no
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

Boot-time wiring, at the same point where you call your chat-hook registration. **Both calls are required.** Pasting the helper function definitions into your module without invoking them leaves the registry empty forever; the symptom is `enumerateOnlinePlayers()` always returning `{}` even with players connected:

```lua
registerChatHook()
presenceRegisterHook()       -- installs the SetAdminCred heartbeat hook
presenceStartRefreshTick()   -- starts the 15-second active refresh
```

`presenceRegisterHook` is what feeds the registry from connect and heartbeat events. `presenceStartRefreshTick` is what keeps entries alive between SetAdminCred bursts (which fire every 7 to 8 minutes in steady state); without it the registry empties out between bursts. If either is missing, the registry stays empty.

Verify the wiring ran by checking `UE4SS.log` after boot for `Presence heartbeat hook registered (SetAdminCred)` and `Presence refresh tick started (15s interval)`. If those lines are absent, the wiring code did not execute.

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

**180-second expiry.** With a 7–8-minute heartbeat cadence, the TTL is NOT the primary eviction mechanism — the 15-second active refresh tick is. The TTL is a safety net for cases the refresh tick misses (e.g. crashed game-mode lookup, mod reload mid-tick). In practice, dead players are typically evicted within 15–30 seconds via the refresh tick's `GetControllerBySteamId` returning nil, well before the 180s TTL fires.

**Inline eviction on controller-gone.** If `gm:GetControllerBySteamId(steam)` returns nil, the player is truly gone. Evict immediately instead of waiting the 180-second expiry. Cuts eviction latency in practice.

## Disconnect signals: the Blueprint-override logout hook fires

An earlier version of this section claimed no disconnect hook fires and heartbeat-expiry was the only eviction path. That was wrong, and the reason it was wrong is instructive: the hooks were on the wrong path. The **native** game-mode logout functions do stay dark from a Lua `RegisterHook` — `/Script/Engine.GameModeBase:K2_OnLogout`, `TIPlayerController:Logout`, `TIGameModeBase:PrintLogout`, and `SafeLogout` all register cleanly and never fire — but the **Blueprint-override** path does fire:

- `/Game/TheIsle/Core/GameModes/BP_SurvivalGameMode.BP_SurvivalGameMode_C:K2_OnLogout` fires from Lua on **every** logout — measured on both a hard/combat disconnect and a completed safe-log (and, being the universal "player left" path, a menu logout too). The parameter is the exiting `AController`; read the steam with `ExitingController:GetSteamId():ToString()`. That read is safe (the pawn is already gone by logout, so don't touch it). It lets you evict the instant the player leaves instead of waiting on the refresh tick or the TTL.

Safe-vs-combat log is a correlation, not a directly readable flag. `/Script/TheIsle.TIPlayerController:PrepareSafeLogout` fires on safe-log **initiation** only (twice per initiation); the BP-override `K2_OnLogout` then fires ~60 seconds later on completion. So on a `K2_OnLogout` fire, check whether a `PrepareSafeLogout` for that steam arrived in the last ~60–65 s → safe log; otherwise → combat/hard log. (The native `TIPlayerController:Logout` carries a `bWasSafeLog` bool, but that hook stays dark from Lua, so the correlation is how you recover the distinction.)

The server console also logs `LogTheIsleJoinData: [...] Left The Server while not being safelogged, ...`, but that is an internal C++ log line, not a hookable UFunction — the BP-override `K2_OnLogout` is the Lua-reachable equivalent. Note the sample cookbook above deliberately wires only the heartbeat + 15-second refresh + TTL (that combination is self-sufficient and was the original design). The BP-override logout hook is an **event-driven upgrade** you add on top: register it, and on each fire evict that steam immediately for instant, event-driven eviction plus safe-vs-combat classification — with the heartbeat/TTL still underneath as the safety net for anything the hook misses (crashed game-mode lookup, mod reload mid-tick).

## Verified UFunctions on the current build

These hook successfully via `RegisterHook` and actually fire during normal play:

- `/Script/TheIsle.TIPlayerController:SetAdminCred(bool)` is the universal heartbeat. Steady-state cadence is bursts of two fires ~14 seconds apart, with bursts spaced ~7–8 minutes apart (verified 2026-05-18 after correcting an earlier "~90 seconds" observation that turned out to be inflated by connect/reconnect re-evals in a noisy test). The 15-second active refresh tick (not the heartbeat hook) is what keeps the registry fresh between bursts.
- `/Script/TheIsle.TIPlayerController:SetDevCred(bool)` fires alongside SetAdminCred. Could also serve as heartbeat.
- `/Script/TheIsle.TIPlayerController:PrepareSafeLogout()` fires when safe logout starts. Fires twice per initiation.
- `/Script/TheIsle.TIPlayerController:GetChatMessage(...)` fires when chat is received. One fire per receiver in range, hence the dedup requirement on chat handlers.

These hook successfully but never actually fire in normal play (they exist as declared UFunctions but no game code path triggers them). The set below is composite — primary backing is OnLoginProbe + DamageProbe + ParkRedeemProbe source code, plus some observational play. Probe-anchored items are confirmed; observational items are marked.

Probe-anchored (verified by source code in `Mods/<Probe>/Scripts/main.lua`):
- `EndLoadingScreen`, `OpenSpawnZoneSelect`, `OpenFactionSelect`, `Logout`, `SafeLogout`, `CancelSafeLogout`, `ClientLogout`, `ClientKicked`, `RequestInitialSpawn`, `SpawnInRandomNest`, `RequestSpawnInNestAsSpecies`, `RequestSpawnInNestByKey` — all in OnLoginProbe's candidate registration list.
- `OnPawnDeath` — in DamageProbe's candidate registration list.
- `OnPlayerRespawned`, `RequestRespawn`, `AddSpawnRequest`, `TryToRespawn`, `ClientRestart`, `Possess` — in ParkRedeemProbe's candidate registration list.

Observational only (not in any probe source I located, but reported as non-firing during normal play):
- `OnPlayerLoginStatusChange`, `UpdatePlayerCredential`, `PrintLogout`, `OnPawnKicked`, `RequestSpawnInNest`

These did not register on the class paths I personally tried (this is "didn't work for me at the paths I tested", not "impossible" — if you have a working hook path for any of these, please open an issue or PR):

- `K2_PostLogin`, `PostLogin`, `HandleStartingNewPlayer`, `GenericPlayerInitialization`, `OnPostLogin` on every game mode candidate. (Note: `K2_OnLogout` is *not* in this dead list — it registers and fires on the **Blueprint-override** path `BP_SurvivalGameMode_C:K2_OnLogout`; see "Disconnect signals" above. Only the native `/Script/Engine.GameModeBase:K2_OnLogout` stays dark.)
- `ReceivedPlayer`, `K2_OnDestroyed`, `ServerInitConnection` on every player controller candidate

The lesson: do not assume a UFunction by name. Build a spy mod that hooks every plausible candidate first, run normal play, observe which ones fire. Some of these may be reachable through a class path I didn't try; others may be dead code paths from the game's perspective.

## Closing notes

This is a high-leverage pattern in the EVRIMA Lua modding toolkit, and the one I trust for "what players are online" in production because its disconnect handling is proven. The engine collections (`gm.AllPlayerControllers` as a TSet via `ForEach`, `GameState.PlayerArray` as a TArray) also give you an online snapshot directly, and for a one-shot read they are less code; reach for the registry specifically when you need eviction you can rely on, until someone verifies the collections' disconnect behavior at scale.

A common early mistake is implementing only the heartbeat hook without the 15-second refresh tick. The mod works for the first connection burst then goes dark for 4 minutes between heartbeat bursts. The refresh tick is what makes the pattern usable in production.
