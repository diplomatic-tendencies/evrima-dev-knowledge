# EVRIMA 0.21.720 migration notes for mod and server owners

The 0.21.720 update (the skin-system overhaul patch) is the first patch since these docs went up that materially breaks documented recipes. This is the triage list: what broke, what survived, and the gotchas of the update process itself. Everything here was hit and verified on real servers (a dev box and a large production server) during the week after the patch.

## The headline: what actually broke

**One recipe broke: the customizer write.** `SetCustomizerData` — which was a validated Server RPC before the patch too; its declaration is identical in pre- and post-patch dumps — now silently drops server-originated writes. Every skin mod built on the old mutate-and-replicate recipe reports success and does nothing. The likely mechanism is the patch's new `ValidateAndSanitizeColors` step (attributed, not traced — the customizer doc lays out the evidence honestly). The replacement recipe (direct property write + `ForceNetUpdate`), the grown struct (four new fields including three new paintable regions), and the new `PatternIndex` validation trap are all in the updated [EVRIMA_Customizer_Field_Map.md](EVRIMA_Customizer_Field_Map.md) — if you only read one thing after this page, read that.

**Every recipe we re-tested survived.** Re-verified live on 0.21.720:

- The state-restore setters (`SetGrowth`, vitals, mutations, elder stacks, prime) all still exist and work — a full restore including quest-mutation unlocks and entombed stacks round-trips correctly
- The prime/elder force recipe works
- Chat hooks (`GetChatMessage`) and the C++ `UpdateChat` delivery recipe work
- The presence registry pattern works
- UE4SS **3.0.1 loads fine on 0.21.720** — no UE4SS update needed

That list is exhaustive — it is what was individually re-verified. Everything else in these docs (the AI spawn-pair matrix, the spawnable-actors catalog, the RCON table, the reference material generally) survived loading and smoke testing but has not been re-probed on 0.21.720; per this patch's own silent-failure theme, re-verify a recipe on a dev box before you lean on it.

## Gotchas of the update process itself

These cost real downtime; they are worth more than the rest of this page.

**1. `steamcmd validate` deletes custom files in the install root.** If your `start_server.bat` (or any launcher/config you added) lives next to the server exe, validate removes it — and the server then "won't start" in a way that looks like the patch broke it. Keep copies of anything custom that lives inside the install tree, and expect to restore them after updating.

**2. Run steamcmd twice.** The first pass after a big patch can report success while leaving the install incomplete. A second `app_update ... validate` pass is cheap insurance.

**3. Admins can lose status after the patch — carry the config block under BOTH sections.** One production recovery hit "I'm in Game.ini but not admin" right after updating. The confusing part, reported exactly as it happened: the `AdminsSteamIDs` block was present and unchanged under `[/Script/TheIsle.TIGameStateBase]`, admin was silently dead anyway, and **adding the same block under `[/Script/TheIsle.TIGameSession]` is what fixed it**. Meanwhile the property is still declared on `ATIGameStateBase` in the 0.21.720 dump, and a different box runs admins under only the `TIGameStateBase` section without issue post-patch — so this is not a clean "the section moved" story, and I cannot give you the mechanism. The advice that survives both data points: keep the block under both sections. The duplication costs nothing, and one of the two is the one your server reads.

**4. Map load takes noticeably longer.** ~17 seconds for the world load on our hardware post-patch. If you have watchdogs or scripts that assume boot timing, re-tune them before concluding the server hangs.

**5. The server looking "stuck" in the log right after boot is usually not a bug.** A freshly booted server can sit quiet in the log while being perfectly joinable — the tail of the boot log is not a liveness indicator. Before restarting anything, just try connecting a few times; one post-patch "server is hanging" panic resolved itself on the third connection attempt with no server-side change at all.

## New content with modding surface

- **Kentrosaurus** is playable — species lists, group-cap configs, population caps, and anything else keyed per-species needs a `BP_Kentrosaurus_C` entry
- The customizer's **teeth / mouth / claws regions** are new paintable surfaces (see the customizer doc) — mods can paint them even where the in-game UI doesn't expose them
- `ThemeIndex` and a data-driven theme/section system were added to the skin pipeline; its tables are string-keyed (`ColorsBySection`), which is why field names no longer appear in the UI widget's reflection

## Suggested migration order for a modded server

1. Back up everything custom inside the install tree (launchers, configs, `Mods/`)
2. steamcmd update + validate, twice
3. Restore your custom files; make sure the admin block exists under both `[/Script/TheIsle.TIGameStateBase]` and `[/Script/TheIsle.TIGameSession]` (gotcha 3)
4. Boot with offset-dependent C++ mods disabled; regenerate SDKs and rebuild them before re-enabling
5. Patch skin mods to the new customizer recipe before re-enabling them — they fail silently, not loudly, so "it loads fine" proves nothing
6. Re-verify each mod behaviorally, one at a time, on a dev box before prod

The silent-failure theme runs through this whole patch: the RPC that succeeds and does nothing, the pattern index that eats an apply without an error, the validate that deletes your launcher and lets the server "fail to boot." Assume nothing survived until you have seen it work with your own eyes.
