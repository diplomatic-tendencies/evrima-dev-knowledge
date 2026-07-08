# Enabling a cut / withheld playable dino on a dedicated server

Some playable dinos ship in a patch but are not spawnable on a normal dedicated server — the species exists in the game, players who updated already have all of its assets, but it does not appear as a working entry in the spawn menu. Baryonyx and Oviraptor were both in this state after the 0.21.720-era content drops. This is how to make one of them a fully playable, spawnable species on your server **without a client mod** — every updated client already has everything it needs; the work is entirely server-side.

## Verification status (read this first)

Both **Baryonyx** (carnivore) and **Oviraptor** (omnivore) were taken end-to-end on a dev server: spawned from the menu, played with real stats, mesh, animations, customizer/skin, and species abilities (Baryonyx's directional bite), killed, and respawned, with the server stable across the session. That much is live-verified.

What is **not yet verified** is the multi-client replication soak: a second player standing next to the enabled dino, confirming it renders correctly on their machine and nobody crashes. This is lower-risk than the manual-AI-spawn crash class (see [EVRIMA_Client_Mod_Feasibility.md](EVRIMA_Client_Mod_Feasibility.md) and the AI replication notes) because the dino is spawned through the game's *own* player-spawn path, not a hand-rolled `SpawnActor`+`Possess` — it is a legitimate playable, so the engine treats it like any other. But "legit path" is not "tested," and the deploy adds a new proxy DLL (see step 4), so do the two-client check on a dev box and deploy to production in a maintenance window, not blind.

## Why this is even possible: the two cooks differ

The key insight is that the game is cooked twice, differently, and the dedicated-server cook is the one that's missing pieces.

- The **client cook** ships the fully assembled playable at `/Game/TheIsle/Core/Characters/Dinosaurs/<Species>/`: the `BP_<Species>` blueprint, its anim blueprint `ABP_<Species>` (+ physics), the `Attributes/ATT_*` set, the attribute-curve and balance DataTables, the gameplay-ability and montage DataAssets, camera curves, and vocals — plus the full animation set and mesh/materials/skin palettes at the legacy `/Game/TheIsle/Characters/Dinosaurs/<Species>/` path.
- The **dedicated-server cook strips the assembled `/Core/...` blueprint and the anims**, keeping only the raw art at the legacy path plus the native `ATI<Species>` C++ class.

So on the server, `BP_<Species>_C` resolves as a generic `UObject`, not a `BlueprintGeneratedClass`. The soft class pointer that the spawn menu's class list holds can't resolve to a real playable class, so the menu shows a hollow zero-stat "ghost" and the spawn never fires. **No server config or list edit fixes this** — the class physically isn't in the server's asset set. You have to put it there.

That is the whole method: take the assembled blueprint from the client cook, ship it to the server in a `.pak`, and register it into the live class list at runtime.

## The pipeline

Five steps. Each has been run end-to-end for both species.

### 1. Extract the species from the client cook

The client install has everything. Extract just the one species' asset tree with [retoc](https://github.com/trumank/retoc) (tested on v0.1.5), converting the client's zen-format paks to loose legacy assets:

```
retoc --aes-key <AES_KEY> to-legacy "<client-install>\Content\Paks" <out-dir> --filter <Species>
```

- `<client-install>` is your updated The Isle **client** install (the default is under `steamapps\common\The Isle`), not the server.
- `--filter` matches on the asset **path**, so it catches everything in the species folder — including oddly-named assets you wouldn't think to list.
- `<AES_KEY>` is the game's current pak encryption key. **It rotates every patch.** Dump the current one with `AESDumpster` (drag the client executable onto it, or pass the exe as a command-line argument — not on stdin) and take the highest-entropy candidate it prints. Do not hardcode a key from an old build; a stale key silently fails to decrypt.

### 2. Delete the `Sound` folder before repacking — this one is a hard crash

**Before you repack, delete every `Sound` folder from the extracted tree.** retoc's zen→legacy→zen round trip corrupts `SoundWave` assets. A corrupted sound wave throws `ObjectSerializationError ... Bad name index N/3` during async loading, which is a **fatal crash the instant the pawn spawns**.

The trap is that it's deceptive: the character-creator screen does *not* load the dino's sounds, so everything looks perfect right up until the player hits the spawn button — then the full spawn loads the sounds and the server dies. If you're testing and the creator works but spawning crashes, this is almost certainly why.

Dropping the sounds costs nothing on a dedicated server: the server renders no audio, the missing sound imports are nulled gracefully (no crash), and clients still hear the dino from their own official copy of the assets. Mesh, materials, animations, and DataTables all round-trip through retoc cleanly — audio is the only asset type that breaks.

### 3. Repack into a zen pak

Convert the (now sound-free) tree back to a shipping zen pak:

```
retoc --aes-key <AES_KEY> to-zen "<out-dir>\TheIsle" <mod-name>.utoc --version UE5_6
```

This produces the `<mod-name>.pak` / `.ucas` / `.utoc` trio. Use `UE5_6` — the engine version must match the game.

### 4. Deploy the pak and the signature bypass

Drop the three pak files into the server's `TheIsle\Content\Paks\`. On their own they will be **rejected**: the server enforces pak signing, and an unsigned mod pak logs `Couldn't find pak signature file ... pak is invalid` and is ignored.

To load an unsigned pak you need a signature-enforcement bypass — [UniversalSigBypasser](https://github.com/rm-NoobInCoding/UniversalSigBypasser) (the "Universal Signature Bypasser," UE5.6-confirmed) or equivalent. Install its `dsound.dll` proxy plus the `.asi` into `Binaries\Win64\`. The `dsound` proxy coexists fine with UE4SS's own `dwmapi` proxy, so this does not disturb an existing UE4SS install.

> Be honest with yourself about what this is: you are disabling the server's pak-signature check so it will load a pak you built. That's a legitimate thing to do with content **you** extracted and repacked for **your own** server, and it is unrelated to client anti-cheat. It is still a server-security control you are turning off — deploy it deliberately, in a maintenance window, and know that it's a new native DLL in your server process.

### 5. Register the class at runtime — from C++, not Game.ini and not Lua

With the pak loaded, the class now *exists* on the server, but nothing references it yet. It has to be added to the game mode's live class list, and the only thing that works is a C++ side mod. The two obvious shortcuts both fail:

- **`Game.ini` `AllowedClasses=<Species>` does not register a mod-pak class.** At boot the game builds its class catalog (`AvailableClasses` / `CookedClasses`) from the base asset registry, which does not include mod-pak assets, so your entry is silently dropped.
- **Lua cannot build the entry.** Lua can grow the `AvailableClasses` array and set plain-old-data fields, but assigning the entry's `TSoftClassPtr Class` field is a native crash that `pcall` cannot catch — it takes the server down.

A small C++ side mod (same toolchain as any other — see [EVRIMA_CPP_Side_Mods.md](EVRIMA_CPP_Side_Mods.md)) does it cleanly. The mechanism:

1. Get the live game mode (`UGameplayStatics::GetGameMode`).
2. **Donor-copy** an existing entry from the live `CookedClasses` list — pick a species with the same diet as the one you're adding (a live carnivore entry for Baryonyx, etc.). Copying a real entry gives you a struct with a valid shape and a valid `Diet`, so you only have to change the class pointer.
3. **Repoint the soft class pointer** on the copy: zero its `Class.WeakPtr` (to force the engine to re-resolve rather than reuse a cached resolution), then set the asset path FNames — `Class.ObjectID.AssetPath.PackageName` to `/Game/TheIsle/Core/Characters/Dinosaurs/<Species>/BP_<Species>` and `.AssetName` to `BP_<Species>_C`. (`UKismetStringLibrary::Conv_StringToName` builds the FNames.)
4. Build the new list as the current `CookedClasses` plus your entry, then call `SetNewAvailableClasses(newList, newList, adminController, /*bIsRcon=*/false)` and broadcast `UpdateClassesList()` so any open spawn menu re-reads.

Two things to know about the runtime call:

- **`SetNewAvailableClasses` needs a live admin controller.** It takes an `AdminController` argument and does nothing useful without a valid one. That means you cannot register on a completely empty server — there's no controller to hand it. In practice: register when a player/admin is online (poll and retry, or trigger it on first join). On an empty server the registration simply no-ops until someone connects. This is the ceiling of "enabled by default": the species becomes available the moment the first player is on, not at boot.
- **Guard the pointer work.** The soft-pointer surgery is the kind of raw memory work where a wrong offset is an access violation, not a catchable error. Run it behind a structured-exception guard so a bad build degrades to "didn't register" instead of "crashed the server."

Result: the catalog and available lists grow by one, the new species shows up in the spawn menu as a real entry, and it spawns and plays.

## The myth that wastes everyone's time

You will find community claims that the blocker is a `LogAssetManager: Ignoring PrimaryAssetType <X> - Conflicts with Characters` warning, and that the fix is to re-cook the blueprint so its primary asset type is `Characters`. **This is a red herring.** That warning appears even for a pak built from completely unmodified official assets, and the dino spawns fully anyway. Re-cooking the blueprint in a fresh Unreal editor is actually what *creates* a wrong asset type; extracting the official cooked asset preserves the correct one. Don't chase this — it is not the blocker.

The two real blockers are the ones this doc addresses: the assembled class is absent from the server (fixed by the pak), and nothing ever calls the register-and-replicate step (fixed by `SetNewAvailableClasses` + `UpdateClassesList`).

## Pipeline gotchas

- **The AES key rotates every patch.** Re-dump it after every game update; a stale key just fails to decrypt with no obvious "wrong key" message.
- **If you extract with a CUE4Parse-based tool instead of retoc, watch the Oodle initialization.** Initializing the Oodle helper with a null path can bind to a decompress-only native Oodle that lacks the compressor, giving an `EntryPointNotFound` at runtime. Point it explicitly at a full `oodle-data-shared.dll` (the small oo2core copies bundled with extraction tools are decompress-only).
- **If your server auto-restarts (a watchdog, a scheduled task), the mod DLL is file-locked while the server runs, and a plain file copy over it silently no-ops** — worse, a copy that preserves the source timestamp makes a DLL that never landed look "old" rather than "not updated," which reads like a phantom revert. Deploy a C++ side-mod DLL by stopping the server fully (it can take 25–30s to release the file handle), copying, then hash-verifying source against destination before relaunch. Don't trust the copy silently.

## Per-species specifics

Both were done with the identical pipeline; the only per-species inputs are the name, the diet, and the size of the extract.

| Species | Diet (`EDietType`) | Extract size | Notes |
|---|---|---|---|
| Baryonyx | carnivore (0) | ~916 packages | Full playable — real stats, customizer, mesh/anims, directional-bite ability |
| Oviraptor | omnivore (2) | ~985 packages / ~290 anims | Full playable — spawn, play, death, respawn all verified |

`EDietType` is `0 = carnivore, 1 = herbivore, 2 = omnivore`; it drives which donor entry you copy in step 5 and how the dino is grouped in the menu.

If you enable more than one, register them all in a single `SetNewAvailableClasses` call rather than one per species — each call replaces the whole list, so batching avoids clobbering.

## Gotcha: the Baryonyx lunge boots the player — cap growth to 75.5%

Baryonyx's pounce/lunge (its `GA_Ambush_C` variant of the shared `GA_PounceReal_C` ability) has a **vanilla blueprint defect** on 0.21.720. It reproduces on completely unmodified servers, so it is not caused by the enablement above or by your pak — but you will meet it the moment players run adult Baryonyx, so it belongs here.

**The symptom:** using the lunge intermittently disconnects the *using* player — a clean, instant boot to the main menu. The client process stays alive, the server stays healthy, and at shipped log verbosity the server logs nothing, which makes it look like a client problem. It is growth-gated: near-adult Baryonyx trigger it while sub-adults lunge fine (I've only seen it past roughly 90% growth — that figure is observational, not something I've pinned exactly).

**Why (dissected from the ability's bytecode; the growth figure is the observational part):** every lunge activation arms a looping ~10 Hz timer that is never cleared and sets a one-way "hit something" latch that is never reset. While that latch is false the leaked timers early-out, which is why the lunge "works several times" first. The first real contact flips the latch on permanently, and from then on the banked timers churn location/rotation and ability replication on that one player's connection until the engine closes the connection over it — the silent boot. Adult size makes the ability's close-detect trip almost immediately, which is why it only bites near full growth.

**A second, server-wide effect worth knowing:** the same ability briefly applies a global time-dilation slow (scaled by the caller's ping) and restores it when the ability ends. If the ability ends abnormally — the boot itself is one such path — that slow can get **stuck server-wide**, leaving everyone in slow motion until something resets it. A high-ping Baryonyx spamming lunge can drag the whole server's tick down, not just crash itself.

**The workaround I use: cap Baryonyx growth at 75.5%.** Holding every Baryonyx pawn at or below `0.755` growth keeps them in the sub-adult band that lunges cleanly, comfortably under the boot threshold — so the ability stays fully usable and nobody gets booted. Enforce it as a ceiling the same way per-species caps are held: a periodic sweep that re-asserts the cap on any Baryonyx pawn that has grown past it (see [EVRIMA_Group_Size.md](EVRIMA_Group_Size.md) for that re-stamp sweep shape). One caveat — `SetGrowth` resets a pawn's vitals to the max of the new size, so apply the cap as a one-shot when a pawn first crosses it rather than every tick, or you will thrash vitals. The tradeoff is that Baryonyx never reaches full adult size on your server.

**Alternatives, if you would rather keep full growth:** strip or disable the lunge ability specifically for adult Baryonyx (keeps the size, loses the adult lunge), or fix the defect at the source — on ability end, clear the leaked timers, reset the "hit" latch, and force global time-dilation back to `1`. The source fix keeps both full growth and a working lunge but is more work, and the client runs its own *predicted* copy of the ability, so confirm in a two-client test that the fix actually stops the boot and not just the server-side half of it.

## Closing notes

The mental model is: the client already has the dino; the server is the one that was handed an incomplete cook. You are not creating content, you are restoring the assembled class the server cook removed, then telling the running game mode about it through its own class-list setter. No client mod, no re-cook, no editor — extract, strip audio, repack, load past the signature check, and register at runtime.

The one thing left before this is production-ready is the two-client replication soak. Everything upstream of that is proven; that check is cheap and is the honest gate between "works for me on dev" and "safe to enable on a live server." See [EVRIMA_Asset_Extraction.md](EVRIMA_Asset_Extraction.md) for the extraction toolchain in general and [EVRIMA_Spawnable_Actors.md](EVRIMA_Spawnable_Actors.md) for what else does and doesn't spawn server-side.
