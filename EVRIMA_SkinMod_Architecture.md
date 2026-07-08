# SkinMod architecture

> **0.21.720 note:** the color-write recipe this mod was built on (mutate + `SetCustomizerData`) broke in the skin-overhaul patch — it silently no-ops now. The architecture below (persistence, auto-restore-on-login, the chat command layer) is all still correct; swap the apply function for the direct-write recipe in [EVRIMA_Customizer_Field_Map.md](EVRIMA_Customizer_Field_Map.md), sanitize `PatternIndex` per the same doc, and retire the pure-black bump (the gotcha it guarded against was re-attributed — see the retraction section there).

SkinMod lets players set arbitrary RGB colors on their dino via chat commands. Colors persist across reconnect, relog, and server restart via per-player JSON files plus an auto-restore-on-login mechanism.

The architectural challenge is not the color writing itself (the customizer field map doc covers that). The challenge is making the colors stick despite the engine's own skin system trying to restore the original skin on every login.

## Command surface

Three commands cover the entire user-facing API:

- `!skin <r> <g> <b>` sets the **four most visible color slots** (BodyColor, MarkingsColor, FlankColor, UnderbellyColor) to the same RGB. Quick test command. Does NOT touch Detail1Color, EyesColor, or MaleDisplayColor — use the per-slot or `all` variants to change those.
- `!skin all <r> <g> <b>` sets ALL seven color slots to the same RGB.
- `!skin <slot> <r> <g> <b>` sets one slot. Slot aliases accepted by SkinMod's parser:
  - `body` → `BodyColor`
  - `markings` / `marks` → `MarkingsColor`
  - `flank` → `FlankColor`
  - `underbelly` / `belly` → `UnderbellyColor`
  - `detail` / `details` / `detail1` → `Detail1Color`
  - `eyes` / `eye` → `EyesColor`
  - `breed` / `display` / `male` → `MaleDisplayColor`
- `!skin reset` reverts to the engine's stored skin (clears the mod's per-player override).

Color values accept either `0.0` to `1.0` floats or `0` to `255` integers; the parser detects which based on the value range.

## The auto-restore problem

The engine's skin system stores the player's chosen skin as a named `SkinCode` FString on `FCustomizerDataBase`. On login and on certain events (respawn, certain quest completions), the engine reads `SkinCode` and applies the named skin's colors, overwriting whatever colors are currently on the customizer struct.

If the mod sets custom colors and the player logs out, the engine sees the SkinCode is unchanged and re-applies the original skin on relog. Custom colors vanish.

The mod cannot prevent this directly. The engine's skin-load path is not Lua-hookable, and `SkinCode` itself is an FString that cannot be safely read or written from Lua (safety rule 2).

The workaround is auto-restore. After every engine-driven skin load (which the mod cannot intercept), the mod re-applies the player's stored colors. The engine wins the first race, then the mod overwrites the result. Net effect: the player sees their custom colors.

## The auto-restore loop

The loop runs from the mod's 3-second poll tick (NOT the presence registry's 15-second refresh — the registry is just what tells the mod who's online). Each tick iterates online players from the presence registry and checks whether each player's current pawn address has changed since the last apply. If yes, re-apply.

The implementation pattern (live SkinMod):

```lua
local lastPawnAddr = {}  -- steam -> last applied pawn address

local function autoRestore()
    local players = enumerateOnlinePlayers()
    for _, p in ipairs(players) do
        local saved = loadSkin(p.steam)
        if saved ~= nil and p.pawn ~= nil then
            local addr
            pcall(function() addr = p.pawn:GetAddress() end)
            local addrKey = tostring(addr or 0)
            if lastPawnAddr[p.steam] ~= addrKey then
                applySkin(p.pawn, saved)
                lastPawnAddr[p.steam] = addrKey
            end
        end
    end
end

LoopInGameThreadWithDelay(3000, function()
    safeCall("autoRestore", autoRestore)
end)
```

Gating on pawn-address-change (NOT a fixed time cooldown) is the trick: the apply fires once per fresh pawn instance and then sits idle until the pawn changes (death, transform, reconnect). This naturally rate-limits the apply without any cooldown timer.

The first apply happens shortly after the engine's skin-load completes. The engine's skin-load on login takes a few seconds; the 3-second poll picks up the new pawn address within one or two ticks and applies. Some players notice the brief original-colors window before the override kicks in; this is intrinsic to the architecture.

## Persistence layout

```
Mods/SkinMod/Saved/
└── skins/
    ├── <steam>.json
    └── <steam>.json
```

The per-player JSON file is a **flat** object keyed by **engine field names** (BodyColor, MarkingsColor, FlankColor, UnderbellyColor, Detail1Color, EyesColor, MaleDisplayColor) with **lowercase** `r/g/b/a` color sub-keys. No `steam`/`updatedAt`/`colors` envelope:

```json
{
  "BodyColor":       {"r": 0.5, "g": 0.8, "b": 0.2, "a": 1.0},
  "MarkingsColor":   {"r": 0.4, "g": 0.7, "b": 0.1, "a": 1.0},
  "FlankColor":      {"r": 0.9, "g": 0.9, "b": 0.9, "a": 1.0},
  "UnderbellyColor": {"r": 0.1, "g": 0.1, "b": 0.1, "a": 1.0},
  "Detail1Color":    {"r": 0.5, "g": 0.5, "b": 0.5, "a": 1.0},
  "EyesColor":       {"r": 0.4, "g": 0.4, "b": 0.4, "a": 1.0},
  "MaleDisplayColor":{"r": 0.6, "g": 0.6, "b": 0.6, "a": 1.0}
}
```

Only fields the player has explicitly set are stored — the file may contain only a subset of the 7 fields. The file is overwritten on every `!skin` command. The `!skin reset` sub-command deletes the file rather than nulling its contents.

## The pure-black gotcha: retracted

Earlier revisions of this document taught a "pure black renders wrong" gotcha as engine fact — `FLinearColor(0, 0, 0)` supposedly read as "unset" in some replication paths — and prescribed a `bumpPureBlack` helper that nudged every zero channel to 0.01 before writing. **That diagnosis is retracted.** The failure it was invented to explain fits a different, since-documented signature (a whole apply silently dropped with the last-good render kept — see the retraction section in [EVRIMA_Customizer_Field_Map.md](EVRIMA_Customizer_Field_Map.md) for the re-attribution and its remaining open test). The bump is also actively harmful: it makes true black unreachable for players. It is being removed from the production mod. If you built on an older copy of this doc, delete the bump and add `PatternIndex` sanitization (per the field-map doc) instead.

## Coordination with other state-restore mods

If your server runs both SkinMod and a state-restore mod that also captures colors (like DinoStorage), the two systems need to agree on the source of truth.

The clean architecture: state-restore captures the player's current colors at `!store` time. On `!redeem`, state-restore applies the captured colors. SkinMod's auto-restore loop then sees the player has a saved override (or not, if the player never used `!skin`) and applies its own override.

There's a brief moment after `!redeem` where the engine's skin-load runs (overwriting the state-restore's color apply) and the SkinMod auto-restore tick has not yet fired. This is the same brief window as on any login; it's a few seconds of "wrong" colors before the override kicks in. The user experience is acceptable.

The dirty architecture (don't do this): both mods race for the customizer write, neither cooperates, the colors flicker visibly. If you're integrating SkinMod with another mod, plan the handoff so only one of them owns colors at any given time.

## Performance notes

The customizer write is cheap (well under 1 millisecond per write). The disk write on `!skin` command is one synchronous JSON write per command. The auto-restore tick is a constant-time check per online player, executed every 3 seconds.

For a 400-slot server with say 100 players online and 50 of them using `!skin` overrides, per-tick (3-second) work is:

- 100 presence-registry reads (cheap controller lookups)
- About 50 file reads (the actual override checks) — only on players whose pawn address changed since last tick
- A handful of customizer writes (only on fresh-pawn detections — pawn-address-change gating means a settled player triggers zero writes)

This is well under 1 percent of the server's tick budget. Not a perf concern.

## Closing notes

The mod is short (about 400 lines of Lua including the persistence and command parser) but it took several iterations to nail. The pure-black gotcha (since retracted — the surprise was real, the diagnosis was not) was the first surprise; the auto-restore-on-login race was the second. The third surprise was that the engine's skin-load path is not hookable from Lua at all, which forced the auto-restore architecture rather than a clean "intercept and replace" approach.

Once the architecture clicked, the mod has been zero-maintenance in production. The presence-registry-driven heartbeat is a clean event source; the customizer field-write is a safe operation; the per-player JSON is trivial to back up and restore.
