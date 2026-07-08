# EVRIMA customizer field map

This is the working recipe for reading and writing player skin colors on The Isle EVRIMA, plus the field-name to UI-label mapping for each color field.

> **Updated for game build 0.21.720 (the skin system overhaul).** The patch broke the old write recipe and grew the struct. If you maintain a skin mod, read the breaking-change section first — your mod is almost certainly doing nothing right now, silently. Everything in this document was re-verified live on 0.21.720 unless marked otherwise, with one honesty note: the post-patch verification ran through a native C++ side mod performing the same direct property write. The Lua transcriptions below use the same fields and calls but have not themselves been re-run on 0.21.720 yet — POD field writes of this shape are long-proven from Lua, so they are expected to behave identically, but run them on your own dev box.

## The 0.21.720 breaking change

Before the patch, the write recipe was mutate-and-replicate: `GetCustomizerData()`, mutate the fields, `SetCustomizerData(cdata)`. That broke: on 0.21.720, calling `SetCustomizerData` server-side "succeeds" (no error, no crash) and applies nothing. Your pcall returns true, your log says the skin was set, and the dino does not change. This is the worst kind of break: silent.

To be precise about the mechanism, because the obvious explanation is wrong: `SetCustomizerData` was *already* declared `UFUNCTION(BlueprintCallable, Server, Unreliable, WithValidation)` before the patch — the declaration is identical in pre- and post-patch dumps, so "it became an RPC" is not what broke it. What the patch did add is a new function in the pipeline: `ATIGameModeBase::ValidateAndSanitizeColors(Character, FCustomizerDataBase&)` exists in the 0.21.720 dump and in no earlier one. The correlation is strong — a new validate/sanitize step appears, and in the same build server-originated writes through the setter stop landing — but I have not traced the call path, so treat the *why* as attributed, not proven. On the *what*, one honesty caveat: the failing test that condemned the setter also carried an out-of-range `PatternIndex`, which the validation section below shows produces the same silent-failure signature all by itself. A clean isolation — the setter with known-valid indices — has not been run. So "the setter path is dead" is probable (the new sanitize step is there, and setter-based tools from other developers broke on the same patch) rather than isolated. What is beyond doubt is the practical half: the direct write below works, verified live.

`GetCustomizerData()` I now treat as suspect rather than proven-broken: I have not caught a failed read live, but post-patch there is no reason to route reads through a wrapper on the same surface that just ate the setter. Read the live property instead — same data, no question mark.

The recipe that works now is a **direct property write plus a replication kick**:

1. Write the fields directly on the pawn's live replicated property: `pawn.CustomizerData`, not the `GetCustomizerData()` wrapper.
2. Call `pawn:ForceNetUpdate()` afterwards. The property is compare-replicated (`ReplicatedUsing=OnRep_CustomizerData`); the ForceNetUpdate pushes the change out and observing clients re-render (replication reaches all observers by design).

No RPC, no validation hook, no wrapper. Verified live on 0.21.720 (native module, per the header note): all color fields render.

## What is FCustomizerDataBase

`FCustomizerDataBase` is the USTRUCT that holds every visual customization for a player's dino. The 0.21.720 overhaul grew it from 0x98 to **0xC8** bytes and added four fields. Current layout (offsets from the 0.21.720 header dump):

| Offset | Field | Type | Notes |
|---|---|---|---|
| 0x08 | `bIsFemale` | bool | |
| 0x0C | `SkinVariation` | float | see the validation section below |
| 0x10 | `PatternIndex` | int32 | see the validation section — this one bites |
| 0x14 | `ThemeIndex` | int32 | **new in 0.21.720** |
| 0x18 | `MaleDisplayColor` | FLinearColor | |
| 0x28 | `MarkingsColor` | FLinearColor | |
| 0x38 | `BodyColor` | FLinearColor | |
| 0x48 | `FlankColor` | FLinearColor | |
| 0x58 | `UnderbellyColor` | FLinearColor | |
| 0x68 | `TeethColor` | FLinearColor | **new in 0.21.720** |
| 0x78 | `MouthColor` | FLinearColor | **new in 0.21.720** |
| 0x88 | `ClawsColor` | FLinearColor | **new in 0.21.720** |
| 0x98 | `Detail1Color` | FLinearColor | |
| 0xA8 | `EyesColor` | FLinearColor | |
| 0xB8 | `SkinCode` | FString | **do not name this from Lua** (safety rule 2) |

The three new color fields are not decorative struct padding: **they are live paint regions**. Verified by writing extreme HDR values to each and observing the render — teeth, mouth interior, and claws each take their own color. The in-game customizer UI is data-driven, so whether players can reach these regions through the UI depends on the species' section tables; a mod can paint them regardless.

You write field names, not offsets, from Lua — the offsets are listed because the struct GREW, and anything that memcpy'd or serialized the struct by size (C++ side mods, save tooling) needs the new size.

## The color fields and UI labels

Each color is an `FLinearColor` with `R`, `G`, `B`, `A` floats, nominally 0.0 to 1.0. Values above 1.0 render as HDR glow and are stable. The UI-label mapping (determined observationally, one loud color per field):

| Struct field | UI label |
|---|---|
| `BodyColor` | body |
| `MarkingsColor` | pattern / markings |
| `FlankColor` | sides / flank |
| `UnderbellyColor` | belly |
| `Detail1Color` | detail |
| `EyesColor` | eyes |
| `MaleDisplayColor` | male display color / breed |
| `TeethColor` | (new region — teeth) |
| `MouthColor` | (new region — mouth interior) |
| `ClawsColor` | (new region — claws) |

## Write pattern (0.21.720)

The recipe in Lua form. Per the header note: the 0.21.720 verification of this exact sequence (direct field writes, then `ForceNetUpdate`) was done from a native C++ side mod; this transcription has not itself been re-run post-patch.

```lua
local function applyCustomizer(pawn, colors)
    if pawn == nil or colors == nil then return false end
    local okCd, cd = pcall(function() return pawn.CustomizerData end)
    if not okCd or cd == nil then return false end

    local function writeColor(field, c)
        if c == nil then return end
        local ok, err = pcall(function()
            cd[field].R = c.R
            cd[field].G = c.G
            cd[field].B = c.B
            cd[field].A = c.A or 1.0
        end)
        if not ok then print("[skin] " .. field .. " write failed: " .. tostring(err)) end
    end

    writeColor("BodyColor",        colors.body)
    writeColor("MarkingsColor",    colors.markings)
    writeColor("FlankColor",       colors.flank)
    writeColor("UnderbellyColor",  colors.underbelly)
    writeColor("Detail1Color",     colors.detail)
    writeColor("EyesColor",        colors.eyes)
    writeColor("MaleDisplayColor", colors.breed)
    writeColor("TeethColor",       colors.teeth)
    writeColor("MouthColor",       colors.mouth)
    writeColor("ClawsColor",       colors.claws)

    if colors.skinVariation ~= nil then
        pcall(function() cd.SkinVariation = math.floor(tonumber(colors.skinVariation)) end)
    end
    -- PatternIndex needs range validation. See the validation section for why
    -- an out-of-range value here silently blocks this ENTIRE apply.
    local pi = tonumber(colors.patternIndex)
    if pi ~= nil then
        pi = math.floor(pi)
        if pi >= 0 and pi <= 2 then   -- 2 here = Tyrannosaurus (3 patterns); the bound is per-species, see the validation section
            pcall(function() cd.PatternIndex = pi end)
        end
    end

    pcall(function() pawn:ForceNetUpdate() end)
    return true
end
```

Same-tick rule still applies: grab `pawn.CustomizerData` fresh every apply, never cache the wrapper across ticks. And the old rule stands — never name `SkinCode`.

## Read pattern (0.21.720)

Read the same live property, not the `GetCustomizerData()` wrapper:

```lua
local ok, cd = pcall(function() return pawn.CustomizerData end)
if ok and cd ~= nil then
    local bodyR = cd.BodyColor.R   -- etc.
end
```

Reads of the POD fields are safe. Skip `SkinCode` entirely, as always.

## PatternIndex and SkinVariation: the validation map

The overhaul gave these two fields **opposite validation regimes**, and getting this wrong produces the silent-failure trap of the patch. Mapped with an eight-probe sweep (each probe carried a unique loud marker color, so "tolerated" vs "rejected" was unambiguous), then cross-checked against the in-game creator:

**`SkinVariation` is completely unvalidated.** Every value tested passed without blocking anything: -833, -1, 0, 7, 100, 100000. It can never break an apply. Floor it to an integer for hygiene and move on. (The old belief that it is a 0.0–1.0 blend float did not survive testing.)

**`PatternIndex` is strictly validated against the species' pattern table, in both directions.** On a Tyrannosaurus (3 patterns): 0 and 2 apply; **-1, 3, 7, and -629 are all silently rejected**. And the rejection is not per-field — an out-of-table `PatternIndex` makes the client abort the entire skin rebuild, so **every color in the same apply is dropped too**, while the server happily holds the replicated values. Your mod reports success, the values are genuinely on the pawn, and nothing renders. If your skin mod "randomly does nothing for some skins" on 0.21.720, look for a bad PatternIndex in exactly those skins.

The valid range is per-species and you can read it straight off the in-game character creator: **valid `PatternIndex` = 0 to (the creator's pattern count for that species − 1)**. The creator's option N is index N−1 — confirmed by selecting the 3rd pattern in the creator and reading/flipping the index live.

Writing a *valid* `PatternIndex` on a live pawn re-renders the pattern immediately — pattern switching works as a live feature, not just at spawn time.

Sanitize rule for any apply layer: skip the `PatternIndex` field unless it is an integer inside the species' known range (skip, don't clamp — clamping applies a pattern nobody asked for). `SkinVariation` just gets floored.


## Persistence: skins do not survive relog, by design

Direct-write skins are runtime state. On relog the engine rebuilds the skin from its own persisted recipe (the `SkinCode` string, which mods cannot write), so your applied colors revert.

The pattern that works is unchanged from the old architecture: the mod owns persistence.

1. On apply, also save the values to a per-steam JSON file.
2. On player connect (presence registry) and on server boot, re-apply the saved values to the live pawn after a short delay (3–5 seconds, letting the engine finish its own skin load).

The player sets a skin once and it survives reconnects and restarts, because the mod re-asserts it every time the engine restores the original.

## Performance notes

A full write (10 colors, both scalars, ForceNetUpdate) is well under a millisecond. Once per state change is all you need.

## Closing notes

Post-overhaul, the customizer is still one of the friendlier surfaces in EVRIMA modding — the patch moved the goalposts (setter path broken, struct grown, PatternIndex now a trap) but the direct-write recipe is simpler than what it replaced, and the new teeth/mouth/claws regions are cool. The two things that will bite you are both silent: the broken setter path (your old mod "works" and does nothing) and an out-of-range PatternIndex (one bad index eats the whole apply). Guard those two and everything else is POD writes.
