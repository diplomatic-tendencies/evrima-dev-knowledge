# SpeciesMixpack architecture

SpeciesMixpack lets configured species pairs form real groups — a Carno running with a Trike herd, with the group nametags and the shared roster — despite the game only allowing same-species grouping. Admins control which pairs are allowed from chat, live.

This is the architecture doc for the shipped mod (a C++ side mod; the reasons it cannot be pure Lua are part of the story). The underlying mechanism — where the same-species gate actually lives, why the server's group functions don't care about species, and the decoy-invite trick — is in [EVRIMA_Grouping_Mechanism.md](EVRIMA_Grouping_Mechanism.md). Read that first; this doc is about how the production mod wraps that mechanism.

## The problem shape, in one paragraph

The same-species restriction is a client-side compiled branch: the invite dies inside the client binary before anything reaches the server. But the server's own group functions (`ServerJoinGroup` and friends) are species-blind, and the friendly call ("Attract", the hold-2 vocalization) replicates to the server regardless of species — exposing both the caller and everyone who hears it. So the mod listens for the friendly call, and when a configured pair is standing together, it stages a native group invite server-side — with one twist (the decoy, below) that makes the hearer's own client accept it.

## Why this is a C++ mod

Two walls, both documented in the safety rules:

1. **`RegisterHook` silently never fires on `ExecuteLongRoarEffect`** (safety rule 14 — this mod is where that rule was paid for). The hook registers cleanly and stays dark. The only thing that catches the friendly call is a global ProcessEvent pre-callback, which is a C++ facility.
2. The invite staging touches native controller state and fires a Client RPC — the class of work that either crashes from Lua or can't be expressed there (see rule 13's family).

The C++ toolchain it stands on is in [EVRIMA_CPP_Side_Mods.md](EVRIMA_CPP_Side_Mods.md).

## Component map

**Hook 1 — the friendly-call listener.** A global ProcessEvent pre-callback with a branch-light hot path: compare the function's FName against a cached `ExecuteLongRoarEffect` name, then check the vocal is `"Attract"` (a field in the roar param struct), and only then capture the caller and hearer pawn pointers and enqueue the invite work to the game-thread marshal. Everything expensive happens off the hot path; for the 99.99% of ProcessEvent traffic that isn't a friendly call, the callback is a single cached FName compare (the Attract check only runs on actual roar calls).

**Hook 2 — the admin chat surface.** A per-function hook on `GetChatMessage` (this one `RegisterHook`-style keying does catch), gated on the `!mix` prefix, deferring to the marshal like everything else.

**The pair matrix.** A plain text config at `Mods/SpeciesMixpack/Saved/config.txt`: `pair trike carno` lines plus scalars (`enabled`, `allpairs`, `cooldown` seconds, `maxdist` units, and `startupdelay` seconds — how long after boot the mod waits before arming the global ProcessEvent detour; 90 is the shipped default; the admin chat hook installs immediately, so `!mix` works from boot). The delay is not a settling nicety: arming the global detour during the fragile early-boot window crashed a production server whenever pairs (or `allpairs`) were configured at boot, and the deferred, lazy arming is the fix for that incident (see the gotchas). Species names are lowercased and sorted on entry, so `pair Trike Carno` and `pair carno TRIKE` are the same pair. `allpairs on` bypasses the matrix entirely (any species may group). Every admin command that changes state auto-saves the file.

**Game-thread marshaling.** Hook callbacks run inside the ProcessEvent detour and must never call ProcessEvent themselves; all engine mutation is enqueued as closures and drained on the game thread. The pawn pointers captured in the hook are treated as hints only — the invite processor **re-resolves both players fresh by steam id** before touching anything (the stale-pointer discipline from the safety rules, applied).

## The invite flow

1. Species-A player holds 2 (the friendly call) near a species-B player. Their client's own invite logic kills the cross-species invite locally — but the call's server-side effect fires anyway, and the listener catches it with both pawns in hand.
2. On the game thread, the mod gates: different species; pair in the matrix (or allpairs); the hearer isn't already in a group (no poaching); a per-player-pair cooldown; an optional distance cap.
3. If all gates pass, it stages a native invite on the hearer's controller — `GroupInviter` pointed at the real cross-species leader — and fires the group-notification popup. The decoy: the popup names the hearer's *own* pawn, so when they accept, their client's same-species check passes and the accept flows through the fully native path. (Why this works and what the popup limitation is: the mechanism doc.)
4. The hearer holds 2 to accept. The native `ServerJoinGroup` reads `GroupInviter` — the real leader — and forms a genuine group: shared GroupId, shared roster, both directions.
5. Housekeeping: unaccepted invites are cleared after ~12 seconds (the staged `GroupInviter` must not dangle if the leader dies or logs out before the accept), and the cooldown map is garbage-collected.

The result is a native group, not a simulation of one — the nametags and the shared roster behave because the engine formed the group itself. (Grouping in EVRIMA gives exactly those two things; there is no group chat channel or friendly-fire change to inherit — see the mechanism doc.)

## Command surface

All admin-gated (`GetIsAdminCred`), from in-game chat:

| Command | Effect |
|---|---|
| `!mix status` | enabled/allpairs/cooldown/maxdist, pair count, startup delay, hook armed state |
| `!mix pair <A> <B>` / `!mix unpair <A> <B>` | edit the matrix (auto-saves) |
| `!mix pairs` | list the matrix |
| `!mix allpairs on\|off` | any-species mode |
| `!mix cooldown <seconds>` | per-player-pair invite throttle |
| `!mix maxdist <uu>\|off` | optional distance gate (off = the call's own audible range is the gate) |
| `!mix on` / `!mix off` | master switch |
| `!mix reload` | re-read the config file |
| `!mix species` / `!mix whoami` | identify helpers for admins setting up pairs |

## Gotchas worth knowing

- **The hearer must be ungrouped.** Invites to anyone already in a group are silently dropped — a deliberate anti-poaching rule, not a limitation.
- **The popup names the wrong dino** (the hearer's own, not the leader) — that's the decoy working as designed; the accept is real. Cosmetic only, players learn it in one use.
- **Timeouts are clean by construction.** An unaccepted invite is cleared by the mod's ~12 s expiry, and the next friendly call fires a fresh one; the cooldown stops call-spam from becoming popup-spam. (Whether the native decline path also clears a hand-stamped invite hasn't been separately verified — the expiry covers that case regardless.)
- **The staged invite is cleared after ~12 s if unaccepted** — the lifetime-tracking gap in the native path (the mechanism doc's biggest gotcha) is handled by expiry, not by tracking the leader's fate.
- **The early-boot window is a real crash class.** The first release armed the global ProcessEvent detour at init, and that crashed a production server during startup — reproducibly, and only when pairs (or `allpairs`) were configured at boot, i.e. only when the detour armed during the heavy early-boot window. The fix is the deferred lazy arm: wait out `startupdelay`, and only install the detour once the mod is actually active (at least one pair configured, or `allpairs` on). If you build anything else on the global ProcessEvent pre-callback, copy the discipline, not just the callback.

## Verification status

Live-verified: the decoy-accept core (v1.0.0) forms genuine bidirectional cross-species groups through the fully native accept path — shared GroupId, shared roster, both directions, confirmed with real players. The boot-crash fix and the deferred detour arming came in v1.0.4, which is the build this document describes. The pair matrix is exactly the policy knob it looks like: a server can allow a herbivore/omnivore mixing table while leaving carnivores same-species-only, or open everything with `allpairs`.
