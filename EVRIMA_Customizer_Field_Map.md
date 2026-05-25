# EVRIMA customizer field map

This is the working recipe for reading and writing player skin colors on The Isle EVRIMA, plus the field-name to UI-label mapping for each of the seven color fields.

The customizer is interesting because the obvious approach (call `RequestRespawn` with a fresh `FCustomizerDataBase`) crashes from Lua (safety rule 4). The approach that works is mutate-and-replicate on the player's live pawn's existing customizer struct.

## What is FCustomizerDataBase

`FCustomizerDataBase` is the USTRUCT that holds every visual customization for a player's dino. It contains:

- 7 `FLinearColor` fields for the color slots
- 1 `SkinVariation` float (controls intra-skin variation effects)
- 1 `PatternIndex` int (chooses the species' pattern)
- 1 `SkinCode` FString (the persistent skin name)

The FString field is the trap. Naming `SkinCode` in Lua crashes UE4SS marshaling (rule 2). Naming any of the POD fields works fine.

## The 7 color fields

Each color field is an `FLinearColor` with `R`, `G`, `B`, `A` floats in the 0.0 to 1.0 range. The field names on the struct, with the UI labels that map to them:

| Struct field | UI label | Notes |
|---|---|---|
| `BodyColor` | Body | Main body color |
| `BackColor` | Back | Dorsal stripe / back |
| `BellyColor` | Belly | Underbelly |
| `AccentColor` | Accent | Stripes, spots, accent markings |
| `PrimaryColor` | Primary | Secondary highlight (despite the name; "Primary" is the UI's "highlight" channel) |
| `SecondaryColor` | Secondary | Tertiary highlight, often spots or claw color |
| `PaintColor` | Paint | Stripes / paint overlay, often the brightest color in a skin |

The UI labels do not match the struct field names cleanly. `PrimaryColor` is not the primary visible color; it's a secondary highlight slot. The mapping above was verified by setting each field to a known unique color value (e.g. pure red on `BackColor`, pure blue on `BellyColor`) and observing which part of the dino changed in the in-game customizer UI.

## Read pattern (verified)

```lua
local function readCustomizer(pawn)
    if pawn == nil then return nil end
    local cdata
    pcall(function() cdata = pawn:GetCustomizerData() end)
    if cdata == nil then return nil end

    local function readColor(field)
        local c = cdata[field]
        if c == nil then return nil end
        return { R = c.R, G = c.G, B = c.B, A = c.A }
    end

    return {
        body      = readColor("BodyColor"),
        back      = readColor("BackColor"),
        belly     = readColor("BellyColor"),
        accent    = readColor("AccentColor"),
        primary   = readColor("PrimaryColor"),
        secondary = readColor("SecondaryColor"),
        paint     = readColor("PaintColor"),
        skinVariation = cdata.SkinVariation,
        patternIndex = cdata.PatternIndex,
        -- DO NOT name SkinCode here. It's an FString and naming it crashes
        -- UE4SS marshaling. Skip it entirely; capture the visual fields only.
    }
end
```

Note the deliberate omission of `SkinCode`. There is no working way to read it from Lua. For a skin-mod that needs to set skins, the SkinCode is irrelevant; you mutate the color fields directly and the result is whatever color combination you wrote.

## Write pattern (verified)

```lua
local function applyCustomizer(pawn, colors)
    if pawn == nil or colors == nil then return false end
    local cdata
    pcall(function() cdata = pawn:GetCustomizerData() end)
    if cdata == nil then return false end

    local function writeColor(field, c)
        if c == nil then return end
        -- Per-field POD writes. cdata[field] is an FLinearColor.
        cdata[field].R = c.R
        cdata[field].G = c.G
        cdata[field].B = c.B
        cdata[field].A = c.A or 1.0
    end

    writeColor("BodyColor",      colors.body)
    writeColor("BackColor",      colors.back)
    writeColor("BellyColor",     colors.belly)
    writeColor("AccentColor",    colors.accent)
    writeColor("PrimaryColor",   colors.primary)
    writeColor("SecondaryColor", colors.secondary)
    writeColor("PaintColor",     colors.paint)
    -- Optional: skinVariation and patternIndex
    if colors.skinVariation ~= nil then cdata.SkinVariation = colors.skinVariation end
    if colors.patternIndex ~= nil then cdata.PatternIndex = colors.patternIndex end

    -- Push the struct. The flag name and exact semantics of the second arg
    -- vary by build; pass true for bForceReplication.
    local ok = pcall(function() pawn:SetCustomizerData(cdata, true) end)
    return ok
end
```

The pattern is: get the live struct, mutate POD fields by name, pass the same struct wrapper back via the setter. The engine handles replication to the client. The skin updates within a frame or two.

## The pure-black gotcha

The customizer has a subtle bug where pure black `FLinearColor(0, 0, 0)` does not always render correctly. The engine appears to treat pure-zero as "unset" in some replication paths, so a player who set every color to pure black would see a partially-default dino on respawn or relog.

The workaround is to bump any pure-zero channel up by a tiny amount (0.01 is enough to be invisible but defeats the sentinel check):

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

Run this on every color before the write. The visual difference between 0.0 and 0.01 is imperceptible in-game; the difference between "displays" and "vanishes on relog" is everything.

## Pattern index and skin variation

`PatternIndex` (int) selects the species' pattern. The value range depends on species; some species have 4 patterns, others have 6 or 8. Setting an out-of-range index produces a default-pattern dino.

`SkinVariation` (float, 0.0 to 1.0 range) drives intra-pattern variation. The exact behavior varies by species. For some species it's a stripe width modifier, for others it's a stripe color blend. Setting 0.5 produces middle-of-range variation.

These two fields are POD writes, no special handling needed.

## Coexistence with the engine's skin system

The engine has a separate "skin code" persistence system that bundles colors plus pattern under named skins. Setting customizer fields directly bypasses this system. The visual updates immediately, but the engine's `SkinCode` FString does not change.

The practical consequence: a player who logs out and back in may see the engine restore the original skin from the `SkinCode`, overwriting the custom colors. To make custom colors persistent across login, a skin-mod needs to re-apply the colors on player login (hooked via the presence registry's heartbeat, or via a one-shot tick after detecting a fresh login).

The auto-restore pattern that works:

1. Player runs `!skin red green blue` (or whatever command syntax).
2. The mod applies the colors to the live pawn AND writes them to its own per-steam JSON file in `Mods/SkinMod/Saved/skins/<steam>.json`.
3. On every server boot, and on every player connect detected via the presence registry, the mod re-applies the saved colors to that player's live pawn after a short delay (3 to 5 seconds, to let the engine finish its own skin-load).
4. The engine's `SkinCode` does not change; the colors get overridden every time the engine tries to load the original skin.

This produces a stable user experience: the player sets a color once, and it persists across reconnect, relog, and server restart.

## Performance notes

The customizer write is cheap. A single full write (all 7 colors plus variation and pattern, plus the struct setter) takes well under a millisecond. You can do this every tick if you want to (though there's no reason to; once per state change is enough).

The customizer read is also cheap. The auto-restore pattern reads the saved JSON once per player on connect, then writes the colors to the live pawn. No need to cache anything.

## Closing notes

The customizer struct is one of the cleaner USTRUCTs in the EVRIMA codebase: 7 POD-ish color fields, two scalar fields, and one FString that you avoid naming. The mutate-and-replicate pattern works reliably. The only real gotcha is the pure-black sentinel issue, and that is one helper function.

If you need to ship a feature that lets players customize their dino's colors via chat commands, the recipes above are everything you need. The full mod architecture (chat parsing, persistence, auto-restore on login) is documented in `EVRIMA_SkinMod_Architecture.md`.
