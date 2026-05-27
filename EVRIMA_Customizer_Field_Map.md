# EVRIMA customizer field map

This is the working recipe for reading and writing player skin colors on The Isle EVRIMA, plus the field-name to UI-label mapping for each of the seven color fields.

The customizer is interesting because the obvious approach (call `RequestRespawn` with a fresh `FCustomizerDataBase`) crashes from Lua (safety rule 4). The approach that works is mutate-and-replicate on the player's live pawn's existing customizer struct.

## What is FCustomizerDataBase

`FCustomizerDataBase` is the USTRUCT that holds every visual customization for a player's dino. Per `UHTHeaderDump-TheIsle/Public/CustomizerDataBase.h` it contains:

- 1 `bIsFemale` bool
- 1 `SkinVariation` float (controls intra-skin variation effects)
- 1 `PatternIndex` int (chooses the species' pattern)
- 7 `FLinearColor` fields for the color slots (named below)
- 1 `SkinCode` FString (the persistent skin name)

The FString field is the trap. Naming `SkinCode` in Lua crashes UE4SS marshaling (rule 2). Naming any of the POD fields works fine.

## The 7 color fields

Each color field is an `FLinearColor` with `R`, `G`, `B`, `A` floats in the 0.0 to 1.0 range. The field names on the struct (from `UHTHeaderDump-TheIsle/Public/CustomizerDataBase.h`), with the UI labels they map to:

| Struct field | UI label | Notes |
|---|---|---|
| `BodyColor` | body | Main body color |
| `MarkingsColor` | pattern / markings | Pattern overlay (stripes, spots) |
| `FlankColor` | sides / flank | Lateral / flank color |
| `UnderbellyColor` | belly | Underbelly |
| `Detail1Color` | detail | Accent / detail markings |
| `EyesColor` | eyes | Eye color |
| `MaleDisplayColor` | male display color / breed | Display color (most visible during courtship) |

The UI labels do not match the struct field names one-to-one. The mapping above was determined observationally by setting each field to a known unique color value (e.g. pure red on `MarkingsColor`, pure blue on `UnderbellyColor`) and watching which part of the dino changed in the customizer UI.

Note: there is no `BackColor`, `AccentColor`, `PrimaryColor`, `SecondaryColor`, or `PaintColor` on this struct. Those are not real field names; assignments to them will silently no-op via UE4SS Lua's wrapper.

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
        body       = readColor("BodyColor"),
        markings   = readColor("MarkingsColor"),
        flank      = readColor("FlankColor"),
        underbelly = readColor("UnderbellyColor"),
        detail     = readColor("Detail1Color"),
        eyes       = readColor("EyesColor"),
        breed      = readColor("MaleDisplayColor"),
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

    writeColor("BodyColor",        colors.body)
    writeColor("MarkingsColor",    colors.markings)
    writeColor("FlankColor",       colors.flank)
    writeColor("UnderbellyColor",  colors.underbelly)
    writeColor("Detail1Color",     colors.detail)
    writeColor("EyesColor",        colors.eyes)
    writeColor("MaleDisplayColor", colors.breed)
    -- Optional: skinVariation and patternIndex
    if colors.skinVariation ~= nil then cdata.SkinVariation = colors.skinVariation end
    if colors.patternIndex ~= nil then cdata.PatternIndex = colors.patternIndex end

    -- Push the struct via the single-arg setter. The UFunction signature is
    -- `void SetCustomizerData(FCustomizerDataBase NewCustomizerData)` —
    -- there is no bForceReplication arg. Replication happens via the
    -- standard property path (OnRep_CustomizerData on the client).
    local ok = pcall(function() pawn:SetCustomizerData(cdata) end)
    return ok
end
```

The pattern is: get the live struct, mutate POD fields by name, pass the same struct wrapper back via the setter. The engine handles replication to the client. The skin updates within a frame or two.

## The pure-black gotcha (folklore, partial verification)

There's a long-standing claim that pure black `FLinearColor(0, 0, 0)` doesn't always render correctly — that the engine treats pure-zero as "unset" in some replication paths and a player who set every color to pure black would see a partially-default dino on respawn or relog.

The live SkinMod guards against this with an inline check **at apply time** (NOT at save time — the saved file is allowed to contain `(0, 0, 0)` faithfully):

```lua
-- Inside applySkin, right before writing each color field:
if r == 0 and g == 0 and b == 0 then r, g, b = 0.01, 0.01, 0.01 end
cdata[fieldName].R = r
cdata[fieldName].G = g
cdata[fieldName].B = b
cdata[fieldName].A = 1.0
```

Two important details:

1. The bump triggers only when **all three** RGB channels are zero, not per-channel. `(0, 0.5, 0)` is left alone.
2. The bump happens at apply time, not save time. The on-disk skin file (`Mods/SkinMod/Saved/skins/<steam>.json`) may contain pure `(0, 0, 0)` values; the bump is applied when those values flow into a SetCustomizerData call.

The visual difference between 0.0 and 0.01 is imperceptible in-game; the difference between "displays correctly" and "shows debug pattern" — if the sentinel bug is real — is everything.

**Note**: this is folklore-level verification. SkinProbe has no test for the bug specifically. Live observation 2026-05-26 with `detail1=(0,0,0)` rendered fine, suggesting the bug may not affect every field, or may only trigger on relog rather than during normal play. The bump is cheap insurance regardless, and production SkinMod keeps it.

Single-field `SetCustomizerData(cdata)` writes DO replicate visibly to the client — verified live 2026-05-26 (body color changed from blue to green via SkinMod chat command; verified visually in-game). The doc's earlier claim that the engine handles replication via the standard property path holds.

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

The customizer struct is one of the cleaner USTRUCTs in the EVRIMA codebase: 7 POD-ish color fields, two scalar fields (`SkinVariation` float, `PatternIndex` int), one bool (`bIsFemale`), and one FString (`SkinCode`) that you avoid naming. The mutate-and-replicate pattern works reliably. The only real gotcha is the pure-black sentinel issue, and that is one helper function.

If you need to ship a feature that lets players customize their dino's colors via chat commands, the recipes above are everything you need. The full mod architecture (chat parsing, persistence, auto-restore on login) is documented in `EVRIMA_SkinMod_Architecture.md`.
