# SpeciesGroupCap architecture

SpeciesGroupCap enforces per-species group-size caps that survive respawns, with live admin control from chat. Set the Carnotaurus pack limit to 6, and it stays 6 — across respawns, relogs, and server restarts — without touching Game.ini or rebooting anything.

This is the architecture doc for the shipped mod. The mechanism it stands on — where `MaxGroupSize` lives, how the engine consults it, and why every spawn silently resets it — is in [EVRIMA_Group_Size.md](EVRIMA_Group_Size.md); read that first if you want to build your own.

## The problem shape

The engine's per-species group cap is a live field the server actually enforces (`GeneralSettings.MaxGroupSize` on the character). Writing it works immediately. The catch is that character initialization re-stamps the field to the species default on every spawn, so a one-shot write evaporates the next time anyone respawns. The whole mod is therefore a small loop: hold an override table, and keep re-asserting it against live pawns.

## Component map

**Override table + config.** A flat JSON file at `Mods/SpeciesGroupCap/Saved/config.json`, keyed by short class name:

```json
{ "BP_Allosaurus_C": 8, "BP_Carnotaurus_C": 6 }
```

Command arguments accept either the friendly name (`Allosaurus`) or the full form (`BP_Allosaurus_C`); everything normalizes to the `BP_..._C` form (`normSpecies`: if it already matches `^BP_.+_C$` keep it, otherwise wrap it). Hand-edit the file with the full form.

**The sweep.** Every 3 seconds, iterate online pawns and compare each pawn's live `MaxGroupSize` to the override for its species; write only when they differ. This is the re-stamp recovery: a freshly spawned pawn carries the stock value for at most one sweep interval (~0–3 s) before the override is re-asserted. Enumeration goes through the presence registry (see [EVRIMA_Presence_Registry.md](EVRIMA_Presence_Registry.md)), never `FindAllOf`; every pawn read is gated on `GetAddress()` and pcall-wrapped; the write itself is a plain int — the safest class of field write there is.

**Stock defaults, captured not hardcoded.** The mod doesn't ship a table of vanilla group sizes. The first time the sweep touches a species, it records the live value it found as that species' stock default, which is what `reset` restores. Consequence: if no pawn of a species has been online this session, `get` shows the stock as "?" and `reset` reports "default on next spawn" — the next spawn re-stamps to the true default anyway, so nothing is lost.

**Admin surface.** A `GetChatMessage` hook parses `!sgc` commands, gated to an admin list, with a 3-second per-player-per-message dedup (the hook has been observed firing once per chat receiver, so identical repeats are suppressed while distinct commands pass; see [EVRIMA_Chat_System.md](EVRIMA_Chat_System.md)). Replies go through the safe-notify queue. A file-command lane (`Saved/cmd.flag` in, `CommandBridge/Saved/results.ndjson` out, `[id] verb args` lines) lets a Discord bot drive the same verbs.

## Command surface

All admin-only:

| Command | Effect |
|---|---|
| `!sgc set <species> <n>` | set the override (n ≥ 1), persist to disk, apply to all online pawns immediately |
| `!sgc get <species>` | show override, captured stock default, and the live value if a pawn is online |
| `!sgc list` | dump all overrides |
| `!sgc reset <species>` | remove the override, restore live pawns to stock |
| `!sgc reload` | re-read config.json from disk and apply (pairs with hand-editing the file) |
| `!sgc apply` | force a sweep now |

`set` is fully live: table, disk, and every online pawn in the same tick. No restart, no reload, no waiting for the sweep.

## Semantics worth knowing

**The cap gates entry, not membership.** The cap is consulted when someone tries to join — that half is verified live (the join gate honored a raised cap). Lowering a cap below an existing group's size is *expected* to leave the group intact until players leave naturally: no eviction mechanism has ever shown itself in testing, but that exact scenario has not been explicitly run, so treat it as expected behavior rather than verified.

**Overrides apply to online pawns only, by design.** Offline players get theirs on next login via the sweep. The config is the source of truth; the pawns are just where it gets stamped.

**The 0–3 s re-stamp window.** Between a spawn and the next sweep, that pawn briefly carries the stock cap. In practice this is invisible — a player cannot assemble a group in the window — but it is the honest shape of the sweep approach, and shortening the interval below 3 s buys nothing measurable.

## Production status

Deployed on a large production server with overrides active across most of the species roster, running through the 0.21.720 patch without changes — the mod resolves everything by name through the Lua wrapper, so the update's struct reshuffles didn't touch it (contrast with C++ side mods compiled against a pre-patch SDK, which did need rebuilds — see [EVRIMA_Patch_0.21.720_Migration.md](EVRIMA_Patch_0.21.720_Migration.md)).

New species slot in with one command: when Kentrosaurus shipped in 0.21.720, adding its cap is just `!sgc set Kentrosaurus <n>` on the live server — the species-key normalization and capture-on-first-touch defaults mean the mod needs no code changes for new dinos.
