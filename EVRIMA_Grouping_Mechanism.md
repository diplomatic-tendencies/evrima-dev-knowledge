# EVRIMA grouping mechanism and cross-species groups

This is how groups (herds/packs) work on The Isle EVRIMA, where the same-species restriction actually lives, and the server-side recipes for forming a group across species. The headline result is that cross-species grouping does not need a client mod, despite the restriction being enforced client-side; the engine's own server functions are species-blind, and the trick is reaching them without the client's cooperation.

This is a deep document because grouping has more moving parts than it looks like from the outside, and the cross-species path took a long time to find. If you only want the recipe, skip to "Forming a cross-species group." If you want to understand why the obvious approaches fail first, read in order.

What grouping does in normal play is narrow: it puts a nametag over your group members and registers them in a shared roster. There is no friendly-fire change, no shared anything else. Keep that scope in mind; it makes some of the design decisions below (why a one-way roster paint is sometimes good enough) make sense.

## The data model

Three stores hold a group, and they are not the same thing. Conflating them is the first mistake.

- `TICharacterBase.GroupId` is an int32 on the pawn, replicated, with `OnRep_GroupId`. Setters are `SetGroupID(int32)` and `SetupGroupId()`. This is a key, not membership. Two pawns with the same GroupId are in the same group; that is all it asserts.
- `TICharacterBase.GroupMembers` is a `TArray<ATICharacterBase*>` on the pawn — the actual roster.
- The game mode keeps the authoritative group record: an id, a SteamId-to-name map of members, a leader, and a descriptive class label (one species string per group, bookkeeping only).

The critical fact, and the one that breaks the naive "already grouped?" check people write: **every connected player has a unique, non-zero GroupId.** The engine assigns one at spawn (a `GenerateNewGroupId` path). A solo player is not GroupId 0; they are (in one live capture) GroupId 1400550544, with a roster of one. There is no sentinel "ungrouped" value. So a test like `if (a.GroupId == b.GroupId) alreadyGrouped` is correct, but `if (groupId == 0) ungrouped` is always false and useless. To decide whether a pawn is grouped *with anyone*, read its `GroupMembers` roster and look for an entry other than itself.

## The native invite flow, and where the gate is

A player holds 2 near another player to invite them; if the target is the same species, they get a popup and hold 2 to accept, or 3 to decline. The functions involved, on `TICharacterBase` unless noted:

- `InteractGroupInvite(bool bAcceptInvite)` — the hold-2 handler, running on the owning client. Behaviorally the hold-2 *key* covers both halves of the handshake (invite when nothing is pending, accept when something is), and this function is its named handler; how the invite half relays to the server is an unidentified path — a cross-species hold-2 leaves zero server-side trace, which is the core problem this document works around.
- `ReceiveGroupInvite(ATICharacterBase* Inviter)` — registers the pending-invite state. BlueprintCallable.
- `TIPlayerController:Client_GroupNotification(ATICharacterBase* Inviter)` — a Client RPC that paints the accept/decline popup on the target's screen.
- `TIPlayerController:ServerJoinGroup(JoiningCharacter, bIgnoreGroupSize, bIgnoreDistance, bComesFromJoinRequest, FriendSteamID)` — the accept. This is the one server-authoritative touchpoint in a working invite.
- `TIPlayerController:ServerDeclinedGroup` — the decline.
- `TIGameModeBase:AddNewGroupMember(Character, NewGroupId, bIgnoreGroupLimit)` — the game mode's exposed roster mutation. One asterisk on it: a read-only hook on this function stayed silent while a real 4-member group formed, so it is not (observably) on the live native join path. It works when you call it yourself (option 1 below); hook `ServerJoinGroup` if you want to watch real joins.

Controller-side invite state lives on `TIPlayerController`: `GroupInviter` (an object property pointing at the pending inviter pawn), `bIsInviteActive`, `bCanReceiveGroupInvites`.

Here is the part that matters and that cost the most time to establish. **The same-species check is client-side, inline, and not a UFunction.** I grepped the entire object dump for any reflected predicate — `CanGroup`, `CanInvite`, `SameSpecies`, `CompatibleSpecies`, `IsEligible`, anything with a grouping-and-species shape — and there is nothing. There is no config bool either; nothing in the game session, game state, or game mode defaults relaxes it. Where exactly the compiled branch lives I have not pinned to an instruction: the behavioral evidence below proves both halves of the handshake die on the client, the accept-side check demonstrably reads the inviter the popup was handed (that is precisely what the decoy in option 3 exploits), and the most plausible home is the client-local `InteractGroupInvite` handler — but that last step is inference from behavior, not a trace through the binary.

I confirmed this from both directions with a server-side process-event watch:

- When a player holds 2 to invite a *different*-species target, nothing fires server-side at all. The invite never leaves their machine.
- When a player accepts a cross-species invite (after the popup is forced onto their screen by other means, see below), `ServerJoinGroup` does not fire. The accept is also killed client-side before any network call.

So both ends of the handshake are gated on the client, before anything reaches the server. You cannot unlock this with a server mod in the obvious sense, because there is nothing server-side to unlock. The masking approach also fails: rewriting a pawn's `GeneralSettings.ClassName` server-side does not fool the check — I tried it; it has no effect. (Why the mask fails is not discriminated: the client may check the real replicated class rather than the name field, or — the simpler explanation — `GeneralSettings` is not a replicated property, so a server-side write to it never reaches the client at all. Either way, name-masking is a dead end.)

What is true, and what the whole cross-species path rests on, is that the *server* functions do not care about species. `ServerJoinGroup` has bypass flags for group size and distance but no species parameter and no internal species check. `AddNewGroupMember` is a plain roster write. The gate is entirely the client deciding whether to call them.

## Forming a cross-species group: three options

### Option 1 — force it with AddNewGroupMember

The bluntest path. Call the game mode's `AddNewGroupMember(targetPawn, leaderGroupId, true)` and the target is painted into the leader's group. It works across species immediately because it is just a roster write. The nametag appears.

The limitation is that it is a one-directional paint of the game-mode roster, not the full bidirectional join the native accept performs, and the engine re-stamps a fresh GroupId on the target at its next spawn, so the membership does not survive a respawn without a re-apply sweep. For an admin "put these two in a group" command it is fine. For anything player-driven it is the fallback, not the goal.

### Option 2 — drive ServerJoinGroup directly

`ServerJoinGroup` is the real accept, and it is species-blind, so you can call it yourself with server authority and skip the client's decision entirely. The calling convention is the part to get right, and it is not what you would guess.

The native accept fires on the *joiner's* controller, with the joiner's *own* pawn as `JoiningCharacter`, and it reads the joiner controller's `GroupInviter` to decide which group to join. So to join player B into player A's group:

```cpp
// On B's controller, set the pending-invite state to point at A, then accept
// the "proximity" way (bComesFromJoinRequest = FALSE). The server reads
// GroupInviter to pick the target group.
setObjProp(bController, STR("GroupInviter"), aPawn);          // the real leader
setByteBool(bController, STR("bIsInviteActive"), true);
setByteBool(bController, STR("bCanReceiveGroupInvites"), true);

// ServerJoinGroup(JoiningCharacter=bPawn, bIgnoreGroupSize, bIgnoreDistance,
//                 bComesFromJoinRequest=FALSE, FriendSteamID="")
bController->ProcessEvent(serverJoinGroupFn, paramsWith(bPawn, true, true, false, L""));
```

Watch the fourth argument. `bComesFromJoinRequest = true` routes `ServerJoinGroup` into the friend-code / spawn-code join path, where the `FriendSteamID` string is validated as a spawn code rather than a steam id. On a server with spawn codes disabled (`bEnableSpawnCodes = False`, as on the server this was probed on) that path returns "spawn code invalid or used too many times" and nothing happens. Pass `false`, and let the stamped `GroupInviter` carry the target group.

Verified: with the state stamped and `bComesFromJoinRequest = false`, two different-species pawns end up sharing a GroupId, both counted in the group, via the genuine `ServerJoinGroup` accept path rather than a roster paint. This is the cleaner force-group than Option 1 — it is the real bidirectional join — but it still happens without the joining player's consent.

### Option 3 — a native popup the player actually accepts (the decoy)

This is the one that produces the real player-facing experience: the joining player sees a normal group-invite popup and holds 2 to accept, exactly like same-species grouping, and a real cross-species group forms. It works by splitting the two jobs the inviter pawn does.

The client's accept-gate checks the species of the inviter it was handed by `Client_GroupNotification`. The server's `ServerJoinGroup` reads the controller's `GroupInviter` to pick the group. Those are two different values, and they are not forced to agree. So:

- Set the joiner controller's **server-side** `GroupInviter` to the *real* leader pawn (plus the two invite bools). This is what `ServerJoinGroup` will read when the accept fires.
- Push `Client_GroupNotification` with the joiner's **own pawn** as the inviter. The joiner's client now believes it has been invited by a member of its own species (itself), so its hold-2 accept-gate passes and relays `ServerJoinGroup`.

```cpp
// Server pending-invite = the real cross-species leader.
setObjProp(joinerCtrl, STR("GroupInviter"), leaderPawn);
setByteBool(joinerCtrl, STR("bIsInviteActive"), true);
setByteBool(joinerCtrl, STR("bCanReceiveGroupInvites"), true);

// Popup inviter = the joiner's OWN pawn (trivially same-species -> client lets
// the hold-2 accept through).
pushClientGroupNotification(joinerCtrl, joinerPawn);
```

The joiner holds 2, their client sends `ServerJoinGroup` (which it would have refused for the real cross-species inviter), the server reads the stamped `GroupInviter` (the real leader), and joins them to the leader's group. Verified across several species pairs, with the `ServerJoinGroup` accept logging `bIgnoreGroupSize=false` — the genuine native accept, not a forced call.

It works because `GroupInviter` is not force-replicated down over the value `Client_GroupNotification` handed the client. The client's accept-gate reads the inviter from its local popup state; the server reads its own `GroupInviter`. The two coexist long enough for the handshake.

The one cosmetic cost: the popup shows the joiner's own species as the inviter, because that is the decoy you handed it. The accept prompt itself is the real native one; only the name on it is the decoy.

`Client_GroupNotification` is a Client RPC, so push it from C++, not Lua, for the same reason `UpdateChat` is a C++ job (see the chat doc).

## The missing half: a trigger the server can see

All three options put the *forming* of the group in your hands, but they need a trigger — some moment where you decide "invite B into A." For an admin command that is just a chat command. For a player-driven, native-feeling flow, the obvious trigger is "the leader holds 2 near the target," and that gesture is exactly the thing that is invisible to the server: as established above, a cross-species hold-2 invite leaves no server-side trace. I confirmed this the thorough way, with a global process-event watch filtered to the leader's pawn and controller, and a cross-species hold-2 fires nothing at all.

The way through is that the hold-2 key does two things. It attempts the group invite, and it plays the friendly call — a vocalization. The invite is client-gated. The call is a *sound*, and sound replicates to everyone nearby regardless of species. So the call reaches the server even when the invite does not.

`TICharacterBase:ExecuteLongRoarEffect(LongRoarStruct, OwnerCharacter, bIsGroupMember)` fires server-side on each player who hears a long call. Inside `LongRoarStruct`, the `VocalName` field is `Attract` for the friendly call, and the `OwnerCharacter` parameter is the caller. The function fires on the *hearer's* pawn with the caller named, so a single hook hands you both "who called" and "who is standing next to them." It fires for every species, in both directions, confirmed across a croc and a small herbivore calling at each other.

One catch worth its own line, because it sent me chasing ghosts: **`RegisterHook` does not intercept `ExecuteLongRoarEffect`.** The hook installs cleanly and never fires. It is a native call that the string-based `RegisterHook` path does not catch on this build. The global process-event pre-callback does catch it:

```cpp
Hook::RegisterProcessEventPreCallback([this](UObject* Context, UFunction* Function, void* Parms) {
    static FName roar    = FName(STR("ExecuteLongRoarEffect"), FNAME_Find);
    static FName attract = FName(STR("Attract"), FNAME_Find);
    if (Function->GetFName() != roar) return;              // cheap per-call gate
    if (Function->GetParmsSize() < OWNER_OFF + sizeof(void*)) return;   // fail closed
    FName vocal = *std::bit_cast<FName*>((uint8_t*)Parms + VOCALNAME_OFF);
    if (vocal != attract) return;
    UObject* caller = *std::bit_cast<UObject**>((uint8_t*)Parms + OWNER_OFF);
    UObject* hearer = Context;                             // the pawn it plays on
    // -> caller did the friendly call near hearer. Defer to on_update, then run a
    //    cross-species recipe (option 3 for native accept).
});
```

This callback runs on every process-event call server-wide, so keep the hot path to a couple of FName comparisons and bail before touching parameters; do the real work on the next tick. Bound the parameter reads against `GetParmsSize()` rather than trusting the offsets, because a `catch(...)` does not catch an access violation under the default exception model — a future layout change has to fail closed, not fault, given how often this fires. The parameter offsets themselves are build-specific; resolve them through reflection if you want to be safe across updates.

Put together: leader holds 2 near a paired-species player, the call reaches the server through `ExecuteLongRoarEffect`, you read who called and who heard it, and you run the decoy popup (option 3) on the hearer. The hearer holds 2 to accept. Hold-2 to invite, hold-2 to accept, no commands and no client mod, with the leader's "invite" riding on the call sound rather than the gated invite RPC.

## Gotchas

**Stamping `GroupInviter` plants a raw pawn pointer with no native lifetime tracking.** When you set it yourself, you bypass whatever bookkeeping the native invite uses to cancel a pending invite if the inviter disconnects. If the leader dies or logs out before the joiner accepts, the joiner's controller is left holding a pointer at a freed pawn, and a later accept dereferences it. Clear the stamp on a timer if the invite goes unaccepted — set `GroupInviter` to null and `bIsInviteActive` false after a short window, re-resolving the joiner controller live by steam at clear time so you are never touching a stale pointer yourself.

**Never invite a player who is already grouped with someone else.** Because the GroupId value-compare is useless (everyone has a unique one), check the target's `GroupMembers` roster for any member other than itself, and skip the invite if it finds one. Otherwise a cross-species invite can pull a player out of an existing group, corrupting that group's membership. Requiring the target to be solo is the safe rule; a mixed group still grows one solo member at a time.

**The invite bools are byte-aligned on this build, not bitfield bools.** `bIsInviteActive` and `bCanReceiveGroupInvites` sit at distinct byte offsets, so a direct byte write is correct. If a future build packs them into a shared byte as bitfield bools, a raw byte write would clobber neighbors; resolve through `FBoolProperty` and respect the field mask if you want to be safe across updates.

**Pawn pointers from the process-event callback are a frame old by the time you act on them.** Treat them as a hint: read the steam ids, then re-resolve the controllers and pawns live before any write. A pointer whose object slot was reused passes an `IsReal`-style check while pointing at the wrong object.

## Closing notes

The restriction that makes grouping same-species-only is a client-side compiled branch, with nothing server-side to toggle. That sounds like a dead end and is not, because the server's group functions never check species; the client just declines to call them. Reach those functions with server authority and the group forms across species fine.

For an admin "group these two" command, `AddNewGroupMember` or a direct `ServerJoinGroup` is all you need. For a player-driven flow that feels native, the decoy popup gives a real hold-2 accept, and the friendly-call sound gives you a hold-2 invite trigger the server can actually observe. The remaining seam is purely cosmetic: the popup names the decoy rather than the real leader.

The one claim to re-verify across EVRIMA updates is that `GroupInviter` is not force-replicated over the `Client_GroupNotification` value. The decoy depends on it, and a replication change there is the thing that would break it.
