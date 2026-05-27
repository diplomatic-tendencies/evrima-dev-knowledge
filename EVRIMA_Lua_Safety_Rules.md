# EVRIMA Lua native-call safety rules

Every pattern in this document cost at least one server crash and a rolled-back mod version. If you're doing any kind of UE4SS Lua work against The Isle EVRIMA, or against any UE4SS-modded UE5 game with similar struct layouts, this is the cheat sheet you want pinned before you write a single native call.

The reason these matter is that UE4SS Lua native crashes are not catchable with pcall. They typically fire as `EXCEPTION_ACCESS_VIOLATION` somewhere between 30 milliseconds and 30 seconds after the bad call, deep inside the marshaling layer or inside the engine, on a thread that has nothing to do with your Lua code. By the time you see the dump, the call that caused it is long gone. Your only feedback loop is "this pattern crashed, that pattern did not."

Treat every one of these as if it were verified by a fresh-out-of-the-debugger crash dump. Most of them were.

---

## Rule 1: UTISaveManager is poison

Do not call any UFunction on UTISaveManager from Lua. The full list of what crashes:

- `GetCharacterData`
- `PlayerDataToString`
- `StringToPlayerData`
- `SaveToFile`
- `EncryptToHex`
- `DecryptFromHex`

All of them produced 30-second-delayed access violations reading offset 0x70 inside the UE4SS marshaling layer. I burned several iterations trying to use this class for persistence before giving up entirely. If you need persistence, write your own JSON in `Mods/<YourMod>/Saved/` and skip the engine's save infrastructure. The engine save path is not something you can drive cleanly from Lua on this build.

---

## Rule 2: USTRUCT field access depends on what is inside the struct

This is the rule that took the longest to nail down. The naive version of the rule is "USTRUCT field access from Lua crashes." That's wrong. The refined version, after about a dozen experiments:

**Unsafe**: touching FString, pointer, or UObject* fields on a USTRUCT by name. Example offenders: `FCustomizerDataBase.SkinCode` (FString at offset 0x88), every FString field on `FTIPlayerData`, every FString field on `FTIEgg`. Reading or writing these specific fields crashes UE4SS marshaling. Naming the field is what trips the marshal, not just having the struct in scope.

**Safe**: touching POD fields (FLinearColor, float, int, bool) on a struct that also contains an FString. For example, `cdata.BodyColor.R = 0.5` on `FCustomizerDataBase` works cleanly. You can read and write all seven FLinearColors on `FCustomizerDataBase` (`BodyColor`, `MarkingsColor`, `FlankColor`, `UnderbellyColor`, `Detail1Color`, `EyesColor`, `MaleDisplayColor`) plus the `SkinVariation` float and the `PatternIndex` int and the `bIsFemale` bool, as long as you never name `SkinCode` itself. Verified live by writing all seven color fields to fresh values over multiple iterations on the same pawn, with the server staying alive for hours afterward.

**Unsafe**: writing Lua strings into FName fields. `liveStruct.MutationSlot1 = "Truculency"` crashes UE4SS marshaling at 0x70. Same crash for passing Lua strings to UFunctions that expect FName parameters.

**Safe**: writing `FName(s)` userdata into FName fields. `liveStruct.MutationSlot1 = FName("Truculency")` works. UE4SS exposes `FName` as a callable userdata global; the value it returns marshals correctly. Then push the struct back via the relevant `SetXxxStruct(struct, bForceReplication)` UFunction (e.g. `SetReplicatedMutationsData` for `FReplicatedMutationsData`). FName values can contain spaces (e.g. "Accelerated Prey Drive"); validate permissively and only reject obvious garbage like quotes, backslashes, and control characters.

**Safe**: pure-POD USTRUCTs containing only floats, ints, bools, enums, and FNames. `FNutrientsValues` (carb, protein, lipid, etc.) and `FReplicatedMutationsData` (16x FName slots) are both safe to read field-by-field, write field-by-field (using `FName(s)` for FName fields), and round-trip via their respective struct setters.

**Safe**: opaque USTRUCT passthrough in a single UFunction call. `SetX(GetX())` works. The pattern of "get the wrapper, mutate its POD fields, pass back the same wrapper" works too. The mutations propagate to the pawn's live property and replicate to the client.

**Unsafe**: caching the userdata returned by `GetX()` across ticks. Stale-pointer crash about a second later, especially for structs with FString fields. This is what an earlier crash hypothesis got wrong; the original interpretation was "field access crashes" but the actual cause was caching the wrapper, then using it after the source pawn was destroyed.

The rule of thumb that fell out of all of this: in any given tick, get the struct from the pawn fresh, touch only POD fields by name (never FString fields by name), and push it back via the struct setter before the tick ends. Don't carry struct wrappers across ticks.

---

## Rule 3: There is no working bulk-enumeration of online players

Both candidate paths to "give me every connected player" are broken in different ways on EVRIMA dedicated servers.

`FindAllOf("TIPlayerController")` returns stale post-disconnect controllers for a window of a few minutes after a player leaves. Calling `K2_GetPawn`, `GetSteamId`, or really any method on a stale controller crashes the server with a native access violation that pcall cannot catch. Verified the crash twice; the second time I had a pcall around literally every line and still got the AV.

`gm.AllPlayerControllers` does not crash, but it always returns an empty array even when players are connected and actively chatting. I verified this side-by-side with a working chat hook plus `gm:GetControllerBySteamId(steam)` proving the players genuinely exist on the server. The engine just doesn't populate `AllPlayerControllers` on this build. Using this in production looks like "no crashes, no players" which is a particularly nasty silent-failure mode because your mod looks like it's working until you realize nothing fires.

`FindAllOf("TIDinosaurBase")` and `gm.AllPlayerCharacters` are similarly broken. They retain stale ghost pawns that crash on access.

The fix is a per-mod self-maintained presence registry, fed by hooks. The full pattern is documented in `EVRIMA_Presence_Registry.md`. The short version: hook `/Script/TheIsle.TIPlayerController:SetAdminCred` (which fires once per controller on connect and again as a periodic heartbeat), maintain `online[steam] = { firstSeen, lastSeen }` in module scope, and run an active refresh tick every 15 seconds that re-derives controllers via `gm:GetControllerBySteamId(steam)` and evicts entries whose controller goes nil.

Never cache the controller wrapper from any hook parameter. Re-derive it on every iteration via `gm:GetControllerBySteamId(steam)`. This is cheap and always returns a valid pointer for online steams. if anyone knows a better way please add it here.

---

## Rule 3a: FindFirstOf can crash if the class has no live instances

`FindFirstOf("<class-name>")` for a class that genuinely has zero live instances in the world produced a native AV inside `UE4SS.dll` about 1.7 seconds after the call. I tripped this once when probing a nest class while testing a feature where Phase3 reporting showed `rawNests=0`. pcall did not catch.

The fix: gate any `FindFirstOf` on prior evidence that at least one instance exists. The cheapest evidence is whatever code path you used to know the class exists in the first place. If you have nothing better, hold a previously-resolved class pointer in a long-lived module variable from a different code path that does know an instance exists.

---

## Rule 4: RequestRespawn crashes from Lua regardless of arguments

`RequestRespawn(...)` on `/Script/TheIsle.TIPlayerController` always crashes from Lua, no matter what you pass. UE4SS marshaling fails the by-value `FCustomizerDataBase` struct copy because that struct contains an FString field. The crash hits 0x70 inside the struct copy, same as the rule-2 FString crash.

The crash is identical whether you pass nil, a freshly-constructed customizer, a customizer cached from earlier, or whether you call it from a chat-hook context vs a poll-tick context. The function itself is the problem; you cannot reach it from Lua on this build.

Working alternatives:

`controller:RequestSpawnInNestAsSpecies(speciesClass, steamID)` is two params, no `FCustomizerDataBase`. Spawns the player at their owned nest. Requires the player to be on the respawn screen (dead pawn or fresh post-disconnect).

The "transform-in-place" pattern. This is the right answer for any kind of state-restore feature. Don't respawn at all. Apply state (growth, vitals, mutations, nutrients, skin) directly to the player's current live pawn via scalar setters and POD-struct field writes. The workflow that works: `!store` snapshots state and kills the live dino via `SetHealth(0)`; the player respawns naturally as a fresh juvenile of the same species; `!redeem` mutates that juvenile in-place. Full recipe in `EVRIMA_State_Restore_Cookbook.md`.

---

## Rule 5: ClientShowNotification crashes when called from inside a hook callback

`ClientShowNotification` is the standard way to send a chat notification or HUD popup to a specific player. Calling it from inside a hook callback (verified for the `GetChatMessage` hook) produces a synchronous access violation with a deep UE4SS-only callstack. pcall does not catch.

The fix is the defer-action pattern. Queue the work in the hook, drain the queue from `LoopInGameThreadWithDelay`. Notifications fired from a tick callback work cleanly. The same defer-action pattern fixes basically every "I want to react to a hook by doing something heavy" scenario.

The deferred-call pattern in skeletal form:

```lua
local pendingActions = {}
local nextTick = 0

RegisterHook("/Script/TheIsle.X:Y", function(self, ...)
    -- Do read-only work here. Queue the action.
    pendingActions[#pendingActions + 1] = { controller = self, fireAtTick = nextTick + 1 }
end)

LoopInGameThreadWithDelay(3000, function()
    nextTick = nextTick + 1
    local kept = {}
    for _, a in ipairs(pendingActions) do
        if a.fireAtTick <= nextTick then
            local pawn = livePawnFromCtrl(a.controller)
            if pawn ~= nil then
                pcall(function() pawn:SetHealth(0) end)
            end
        else
            kept[#kept + 1] = a
        end
    end
    pendingActions = kept
end)
```

The 3-second window is acceptable for soft-enforce mechanics. For tighter timing, drop the poll interval to 1 second.

---

## Rule 6: Hook parameter wrappers are unstable across ticks

Hooks on `K2_PostLogin` (and several other lifecycle hooks) crash with cached or stale UObject pointers. The hook parameter wrapper is unstable across ticks. Don't cache controller pointers, pawn pointers, or any UObject pointer that came from a hook parameter for use in a later tick.

If you need a stable handle to a player across ticks, store a primitive identifier (steam ID is the obvious choice) and re-derive the controller pointer via `gm:GetControllerBySteamId(steam)` each tick. This is cheap.

---

## Rule 7: Editing Saved/PlayerData/<steam>.sav files mid-session is useless

The engine reads these files only at server boot. Disconnect and reconnect does not refresh; the engine holds player state in memory until reboot. If you want to modify persistent player data, you either need to do it at the live pawn (rule 4's transform-in-place pattern) or you need to mutate the file and restart the server. Mid-session file edits do nothing.

---

## Rule 8: GAS attributes auto-refill when growth changes

This one is subtle and produced more than one bug report from players. Hunger, `FoodValue`, Thirst, Stamina, and Health are `FGameplayAttributeData` properties on the AttributeSet, replicated via `OnRep_*`. They are not plain floats. (Note: the underlying engine field is `FoodValue`, not `Food`, even though common helper names use `food`.) Every `SetGrowth(N)` call recomputes the maximum value for each of these attributes and refills the current value to the new max.

The practical consequence: if you `SetThirst(455)` then later `SetGrowth(0.5)` to apply a growth change, Thirst gets wiped to MaxThirst (which is recomputed from the new growth). Your set is undone.

The fix: always re-apply current vital values after any growth-touching code. The state-restore cookbook documents the right ordering. Briefly, set growth first, then set max-vitals, then set current vitals. The mutation-slot staging (rule applies there too) involves multiple growth steps; vitals get re-applied at the end after the last growth call settles.

---

## Rule 9: UStaticMeshComponent::StaticMesh is not a replicated UPROPERTY

You can spawn an `AStaticMeshActor` server-side, call `SetReplicates(true)`, `SetMobility(Movable)`, `SetStaticMesh(/Engine/BasicShapes/Cube.Cube)`, `SetActorScale3D(...)`, and `ForceNetUpdate()`. Every call returns true from Lua. The client still sees an invisible empty actor.

The reason is that the StaticMesh property on UStaticMeshComponent is not replicated. The server has the cube, the client never receives the assignment. This is a UE engine limitation, not a UE4SS or EVRIMA-specific thing, but it bites server-side modders who try to use it for runtime-spawned visible markers.

The working alternatives:

Spawn a class whose mesh is baked into the class default (the Blueprint asset). For classes like that, "spawn class X" replicates and the client already knows what mesh X has. `BP_Nest_*` classes work, but you have to verify at least one instance exists before calling FindFirstOf on the class (rule 3a). Custom BPs work if you have them. A tiny UE4SS C++ side mod that exposes a multicast or repnotify wrapper around `SetStaticMesh` would also work; the C++ toolchain doc covers this.

The `AStaticMeshActor` plus runtime `SetStaticMesh` combo is a dead end for runtime-spawned visible actors. Do not waste a week on it.

---

## Rule 9a: K2_GetPawn returns a nullptr wrapper when the player is mid-respawn

This one killed a dev server. The crash sequence:

1. A player disconnected and reconnected, landing on the spawn-zone-select screen.
2. The presence registry tick fired its auto-restore code path, which iterated the registry, called `ctrl:K2_GetPawn()` for each online steam, and got back a non-nil Lua wrapper.
3. The non-nil check passed.
4. The wrapper's underlying pointer was actually 0x0; the Lua side just couldn't tell from a `~= nil` check.
5. The first method call on the wrapper (`pawn:GetCustomizerData()`) hit the null deref. AV. Server dead.

`controller:K2_GetPawn()` returns a non-nil wrapper around a null pawn when the controller exists but the player hasn't spawned a dino yet. This happens during the spawn-zone-select screen, mid-respawn between death and species pick, fresh post-reconnect, and a few other states.

The fix is always filter via `pawn:GetAddress() ~= 0`. The wrapper is non-nil; only `GetAddress()` reveals the actual nullptr. Use a helper at every callsite. The proven pattern:

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

The same nullptr-wrapper pattern applies to `world:SpawnActor` (terrain collision returns a wrapper around nullptr) and probably to several other UE4SS-marshalled UObject returns. Treat `pawn ~= nil` as necessary but not sufficient anywhere a pawn comes from an indirect lookup.

Direct chat-hook pawns (the sender is in-game and chatted) don't hit this case because chatting requires a spawned pawn, but the defensive check is still cheap and worth doing.

---

## Rule 9b: K2_DestroyActor on a cached actor wrapper crashes if the actor was destroyed by gameplay

This one was learned twice in the same week, from two different mods.

First crash: a VFX probe spawned a `BP_SmiteEffect`, called the `CustomEvent` UFunction to trigger the lightning animation, the lightning played and the effect actor self-destroyed via the BP's BeginPlay logic. The probe still held a Lua wrapper to the actor from the spawn call. A cleanup command later called `K2_DestroyActor` on that stale wrapper. Native AV crash inside `UE4SS.dll`.

Second crash: an AI probe spawned a Triceratops with a Diabloceratops AI controller, possessed it, the player killed and ate the trike (a normal gameplay event that destroys the corpse actor after a delay). The probe still tracked the spawned pawn. Cleanup ran `K2_DestroyActor` on the freed memory. Native AV.

The mistake in both cases was assuming that a `pcall(function() addr = actor:GetAddress() end)` check would catch the freed-actor case. It does not. Freed-but-not-yet-reused memory still returns a non-zero address. The address check passes, the destroy on that freed slot is what AVs.

The fix is conservative cleanup. Don't try to destroy actors from Lua-side cleanup at all. Clear your tracking table. restart the server if you want full reset of spawned actors. If you really need runtime destroy, you have two options:

Option one: subscribe to a gameplay event that signals "this actor is gone" (PlayerStats HP=0 transition for dinos, or hook the OnDestroyed event if available) and proactively remove from tracking before the gameplay destroy completes. This needs the event to fire reliably, which most don't on this build.

Option two: use a UE-side `IsValid()` or `IsActorBeingDestroyed()` check, but those are not reliably exposed to Lua on this UE4SS build.

The high-confidence pattern for any mod that spawns actors: track for the read path (knowing what was spawned, for logging or stats), don't track for the write or destroy path. Lua-mod-side actor tracking plus Lua-mod-side destroy is fundamentally unsafe when other systems can destroy the same actors.

---

## Rule 10: Server-spawned actors don't replicate by default

Even after `SpawnActor`, the client never gets the actor unless the network flags are flipped. The proven recipe, server-side, on the freshly-spawned actor:

```lua
pcall(function() actor:SetReplicates(true) end)
pcall(function() actor.bAlwaysRelevant = true end)  -- clients receive it regardless of distance
pcall(function() actor:ForceNetUpdate() end)         -- push immediately
```

This is necessary but not sufficient for visible meshes; see rule 9.

---

## Rule 11: AllPlayerCharacters retains stale ghost pawns after !redeem

When a player redeems or transforms via a state-restore mod, the old pawn lingers in `gm.AllPlayerCharacters` with no controller for an indeterminate window. Touching any method on a ghost pawn, including `GetController()` itself, crashes with a native AV.

Filtering by "controller is nil" does not help, because the call to determine nullness is what crashes.

The safe enumeration path is the self-maintained presence registry (rule 3). Both `gm.AllPlayerCharacters` and `FindAllOf("TIPlayerController")` retain stale entries that crash on access; `gm.AllPlayerControllers` doesn't crash but is always empty. The registry pattern from rule 3 is the only viable path.

---

## Rule 12: return must be the last statement in its block

This is a Lua language rule that has nothing to do with UE4SS or EVRIMA, but it has bitten me badly enough to deserve a place here.

If you replace a function body via an editor's find-and-replace, and leave orphan code after the new `return` statement, Lua emits `'end' expected near '<next-token>'` and refuses to load the script.

The bad behavior is UE4SS-specific: when the script fails to load, the mod silently fails. The mod's hooks never register. Its poll loops never start. Reload signals like `Saved/reload.flag` are never consumed because nothing's running to read it. Only a server restart will pick up the fix once you've corrected the syntax error.

Always check `UE4SS.log` for `Error loading script` or `Failed to execute main script` lines after editing mod files. If your mod went dark and nothing fired, that's the first thing to look for.

---

## What is safe (quick reference)

The "always works" list, for quick reference:

Scalar `GetX()` UFunctions returning float, int, bool, or FString.

`pawn:K2_GetActorLocation()` followed by `.X`, `.Y`, `.Z` field reads.

`pawn:K2_GetActorRotation()` followed by `.Pitch`, `.Yaw`, `.Roll` field reads.

`pawn:GetClass():GetFullName()` (a UClass pointer, not a struct).

`pawn:GetIsAlive()` (bool).

`controller:K2_GetPawn()`, with the rule-9a address check.

`controller:GetSteamId():ToString()`.

POD-struct field reads on pure-POD structs: `pawn.NutrientsStruct.CarbValue`, `pawn.ReplicatedMutationsData.MutationSlot1` followed by `:ToString()`.

FName field reads via `:ToString()`. Note: `tostring(fname)` returns "FNameUserdata: 0x..." which is useless; use `:ToString()` to get the real string.

Setter UFunctions for scalars: `SetHealth(100)`, `SetGrowth(0.5)`.

POD-struct field writes for non-FName scalars: `liveStruct.CarbValue = 100`.

POD-struct field writes for FName slots using `FName(s)`: `liveStruct.MutationSlot1 = FName("Truculency")`.

Per-slot UFunction setters with `FName(s)`: `pawn:SetSlot1EquippedMutation(FName("Truculency"))`. Note that these have additional gotchas around growth-staging and batching; see the state-restore cookbook.

Full-struct setters: `pawn:SetNutrientsStruct(struct, true)`, `pawn:SetReplicatedMutationsData(struct, true)`. The second arg is `bForceReplication`.

`FName("Truculency")` to construct an FName userdata. UE4SS exposes `FName` as a callable global. FName values can contain spaces (real mutation names like "Accelerated Prey Drive" do).

`RegisterHook` on UFunctions. The `self` arg is the UObject directly, no `:get()` needed.

`LoopInGameThreadWithDelay(ms, fn)` for recurring tasks. Cancel via `CancelDelayedAction(handle)`.

`controller:ClientShowNotification(FText)`, but only from a poll or timer tick, and only on a freshly-derived controller (not a Lua-cached one from a hook parameter). The safe-notify pattern:

```lua
local function safeNotify(steamId, message)
    local gm = FindFirstOf("BP_SurvivalGameMode_C")
                 or FindFirstOf("TISurvivalGameMode")
                 or FindFirstOf("TIGameModeBase")
    if gm == nil then return false, "no-gameMode" end
    local controller = nil
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

Call this only from a tick callback, never from inside a hook. The fresh `FindFirstOf` plus `GetControllerBySteamId` chain produces a valid pointer with the right lifetime. A stored controller from a hook param does not.

---

## Chat hook signature

`/Script/TheIsle.TIPlayerController:GetChatMessage(NewText, ChatPlayerController, ChatMode, NoFilterMsg)` is the hookable chat entrypoint.

`self` is the receiving controller (the hook fires once per receiver in chat range).

`ChatPlayerController` is the sender's controller. This is what you want for command processing.

Deduplicate on `(sender, msg)` within a 3-second window because the hook fires per-receiver and a server with multiple players nearby will produce duplicate events.

Use a wrapper to extract the underlying UObject from RegisterHook param wrappers; the wrappers are unstable across ticks (rule 6) but valid within the hook callback itself.

The per-sender dedup table grows unbounded on a busy server. Add a periodic TTL GC; 15-minute TTL with GC every 5 minutes works fine in practice.

---

## Spawn UFunction surface (player controller)

These are the player-controller-side spawn entry points and what each actually does. Treat the names as untrustworthy; build a spy version that hooks every candidate and logs which one fires from the normal UI flow before committing to one.

| UFunction | Verdict |
|---|---|
| `RequestRespawn` | Crashes if called from Lua (rule 4). Safe to hook for read-only param inspection. This is the UFunction the normal spawn-zone UI actually triggers. |
| `RequestSpawnInNestAsSpecies` | Safe to call from Lua, two params, no FCustomizerDataBase. Does not fire from the normal UI. |
| `RequestSpawnInNestByKey` | Does not fire from the normal UI. |
| `RequestSpawnFromFriendCode` | Does not fire from the normal UI. |
| `ServerTryToRespawn` | Does not fire from the normal UI. |

Game-mode-side entrypoints:

`/Script/TheIsle.TIGameModeBase:TryToRespawn(controller, steam)` exists. Not tested in production scenarios.

`/Script/TheIsle.TIGameModeBase:RequestSpawnInNest(controller, steam, nest, bPrivateFirst)` exists. Not tested in production scenarios.

The lesson from spending half a week chasing these: build a spy mod that hooks every plausible UFunction first, observe what actually fires during normal play, then commit to one. Several of these are dead code paths from the developer's perspective; they exist in the class but no production path calls them.

---

## Hook to deferred-action pattern

Anywhere you want to react to a hook by calling something heavy (kill, notify, respawn, teleport), defer to a tick. The crash class is the same as `ClientShowNotification` from chat hook (rule 5). The pattern is shown in the rule-5 block above. The 3-second tick is a good default; drop to 1 second for tighter timing requirements.


---

## Closing notes

If you remember nothing else from this document, remember three things.

First, native crashes are delayed. The line of code that caused the crash is not the line in the stack trace, because there is no stack trace; the AV fires deep inside UE4SS or the engine 30 seconds later. Your only feedback loop is "I made this change, server is now alive longer or shorter than before."

Second, USTRUCT field access has nuance. The right rule is "POD field reads and writes are safe; FString and pointer field naming crashes." Not "all struct field access crashes."

Third, the broken enumeration paths are silent failures, not loud failures. `gm.AllPlayerControllers` returning empty is not an error; it's just zero results. Your mod runs, your hooks register, no players ever match. Build the presence registry pattern from day one. It's about 50 lines of Lua and it saves you a week of "why is nothing firing."
