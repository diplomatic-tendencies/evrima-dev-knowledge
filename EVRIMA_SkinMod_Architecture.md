# SkinMod architecture

SkinMod lets players set arbitrary RGB colors on their dino via chat commands. Colors persist across reconnect, relog, and server restart via per-player JSON files plus an auto-restore-on-login mechanism.

The architectural challenge is not the color writing itself (the customizer field map doc covers that). The challenge is making the colors stick despite the engine's own skin system trying to restore the original skin on every login.

## Command surface

Three commands cover the entire user-facing API:

- `!skin <r> <g> <b>` sets every color slot to the same RGB color. Quick test command.
- `!skin <slot> <r> <g> <b>` sets one slot. Slot names: `body`, `back`, `belly`, `accent`, `primary`, `secondary`, `paint`.
- `!skinreset` reverts to the engine's stored skin (clears the mod's per-player override).

Color values accept either `0.0` to `1.0` floats or `0` to `255` integers; the parser detects which based on the value range.

## The auto-restore problem

The engine's skin system stores the player's chosen skin as a named `SkinCode` FString on `FCustomizerDataBase`. On login and on certain events (respawn, certain quest completions), the engine reads `SkinCode` and applies the named skin's colors, overwriting whatever colors are currently on the customizer struct.

If the mod sets custom colors and the player logs out, the engine sees the SkinCode is unchanged and re-applies the original skin on relog. Custom colors vanish.

The mod cannot prevent this directly. The engine's skin-load path is not Lua-hookable, and `SkinCode` itself is an FString that cannot be safely read or written from Lua (safety rule 2).

The workaround is auto-restore. After every engine-driven skin load (which the mod cannot intercept), the mod re-applies the player's stored colors. The engine wins the first race, then the mod overwrites the result. Net effect: the player sees their custom colors.

## The auto-restore loop

The loop is triggered by the presence registry's heartbeat. Every time the heartbeat sees a player (every 15 seconds via the refresh tick), the mod checks if there's a saved color set for that player. If yes, it re-applies the colors.

The implementation needs to be carefully gated:

```lua
local lastAppliedAt = {}  -- steam -> os.time() when last applied
local REAPPLY_COOLDOWN = 10  -- seconds; don't re-apply more than this often

local function maybeReapplySkin(steam, controller)
    local saved = loadSavedColors(steam)
    if saved == nil then return end  -- no override for this player

    local now = os.time()
    if lastAppliedAt[steam] and (now - lastAppliedAt[steam]) < REAPPLY_COOLDOWN then
        return  -- rate limit
    end

    local pawn = livePawnFromCtrl(controller)
    if pawn == nil then return end  -- pawn not yet spawned

    local ok = applyCustomizer(pawn, saved)
    if ok then
        lastAppliedAt[steam] = now
    end
end
```

The 10-second cooldown prevents the mod from re-applying on every 15-second heartbeat (which would technically work but produces unnecessary work).

The first apply happens shortly after the engine's skin-load completes. The engine's skin-load on login takes a few seconds (varies by client connection speed); the mod's 15-second heartbeat plus apply usually catches up within 20 to 30 seconds of login. Some players notice the brief original-colors window before the override kicks in; this is intrinsic to the architecture and not something a Lua mod can avoid.

## Persistence layout

```
Mods/SkinMod/Saved/
├── skins/
│   ├── <steam>.json
│   └── <steam>.json
└── readme.txt
```

The per-player JSON file:

```json
{
  "steam": "76561198XXXXXXX",
  "updatedAt": 1716508800,
  "colors": {
    "body":      {"R": 0.5, "G": 0.8, "B": 0.2, "A": 1.0},
    "back":      {"R": 0.4, "G": 0.7, "B": 0.1, "A": 1.0},
    "belly":     {"R": 0.9, "G": 0.9, "B": 0.9, "A": 1.0},
    "accent":    {"R": 0.1, "G": 0.1, "B": 0.1, "A": 1.0},
    "primary":   {"R": 0.5, "G": 0.5, "B": 0.5, "A": 1.0},
    "secondary": {"R": 0.4, "G": 0.4, "B": 0.4, "A": 1.0},
    "paint":     {"R": 0.6, "G": 0.6, "B": 0.6, "A": 1.0}
  }
}
```

The file is overwritten on every `!skin` command. The `!skinreset` command deletes the file rather than nulling the contents.

## The pure-black gotcha

The engine treats `FLinearColor(0, 0, 0)` as "unset" in some replication paths. A player who set every color to pure black would see partial-default colors on relog. The fix is a `bumpPureBlack` helper that nudges any pure-zero channel to 0.01 (imperceptible but defeats the sentinel check).

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

Apply this to every color before writing. The visual difference between 0.0 and 0.01 is unnoticeable; the practical difference is the colors persisting across login.

## Coordination with other state-restore mods

If your server runs both SkinMod and a state-restore mod that also captures colors (like DinoStorage), the two systems need to agree on the source of truth.

The clean architecture: state-restore captures the player's current colors at `!store` time. On `!redeem`, state-restore applies the captured colors. SkinMod's auto-restore loop then sees the player has a saved override (or not, if the player never used `!skin`) and applies its own override.

There's a brief moment after `!redeem` where the engine's skin-load runs (overwriting the state-restore's color apply) and the SkinMod auto-restore tick has not yet fired. This is the same brief window as on any login; it's a few seconds of "wrong" colors before the override kicks in. The user experience is acceptable.

The dirty architecture (don't do this): both mods race for the customizer write, neither cooperates, the colors flicker visibly. If you're integrating SkinMod with another mod, plan the handoff so only one of them owns colors at any given time.

## Performance notes

The customizer write is cheap (well under 1 millisecond per write). The disk write on `!skin` command is one synchronous JSON write per command. The auto-restore tick is a constant-time check per online player, executed every 15 seconds.

For a 400-slot server with say 100 players online and 50 of them using `!skin` overrides, the auto-restore work is:

- 100 heartbeat reads per tick (cheap controller lookups)
- About 50 file reads per tick (the actual override checks)
- About 5 customizer writes per tick (rate-limited by the 10-second cooldown)

This is well under 1 percent of the server's tick budget. Not a perf concern.

## Closing notes

The mod is short (about 400 lines of Lua including the persistence and command parser) but it took several iterations to nail. The pure-black gotcha was the first surprise; the auto-restore-on-login race was the second. The third surprise was that the engine's skin-load path is not hookable from Lua at all, which forced the auto-restore architecture rather than a clean "intercept and replace" approach.

Once the architecture clicked, the mod has been zero-maintenance in production. The presence-registry-driven heartbeat is a clean event source; the customizer field-write is a safe operation; the per-player JSON is trivial to back up and restore.
