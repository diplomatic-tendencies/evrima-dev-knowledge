# EVRIMA chat system and cross-species delivery

This documents how text chat moves through the engine on The Isle EVRIMA, and the working recipe for delivering a chat line to a recipient the native filter would otherwise skip. The most useful application is making local (proximity) chat carry across species, but the same mechanism covers any case where you want to put a line in a specific player's chat box from a mod.

Two functions carry the whole system. Calling the wrong one from Lua is the first thing that bites, so start there.

## The two functions

`TIPlayerController:GetChatMessage` is the send-side surface. You hook it to read what players type; this is how every chat-command mod intercepts `!commands`. One thing the name hides: in production this hook has been observed firing once per *receiving* controller in chat range — `self` is a receiver, and the sender is the `ChatPlayerController` parameter. So take the sender from the parameter, never from `self`, and dedup by (sender, message) within a short window (a few seconds), or one spatial `!command` sent near five same-species players executes five times. Parameters:

- `NewText` (FText) — the message
- `ChatPlayerController` (TIPlayerController) — the sender's controller
- `ChatMode` (uint8, EChatMode)
- `NoFilterMsg` (FText) — a long-standing fourth parameter (it is in dumps from well before the 0.21.720 patch). The name suggests an unfiltered copy of the message, but I haven't verified what populates it. Hooks that resolve the first three parameters by name can ignore it.

`TIPlayerController:UpdateChat` is the deliver path, and it is a Client RPC. The engine calls it on a recipient's controller to make a line appear in their chat box. Parameters:

- `Sender` (FText) — the display name
- `Text` (FText) — the line itself
- `SenderSteamId` (FString)
- `ChatMode` (uint8, EChatMode)
- `bIsDev`, `bIsAdmin` (bool)

`EChatMode` is `{ Spatial = 0, Global, Admin, Logging }`. Spatial is local proximity chat, Global is server-wide, Admin is the admin channel, Logging is what it sounds like. `bEnableGlobalChat` in the game session config gates the Global tab; it has been true on every server I have checked, but it is an operator-settable flag, so verify yours. With it on, the Global tab is server-wide and already crosses species; the interesting work is all in Spatial.

To read chat you hook `GetChatMessage`. To inject or re-deliver chat you call `UpdateChat` on the target controller. The read side is safe from anywhere. The deliver side is not, and that distinction is the whole reason this document exists.

## The deliver side crashes from Lua

Calling `UpdateChat` from UE4SS Lua takes the server down. It is an access violation on the native side, inside the RPC's FText serialization, and `pcall` does not catch it because the fault is below the Lua boundary; the process is simply gone.

The cause is the FText marshaling. UE4SS Lua hands the native RPC an FText whose internal shared reference is not what the serializer expects, the serializer dereferences it during replication, and it faults. How far that generalizes to other FText- or FString-bearing Client RPCs is inference, not a tested rule — and there is a proven-safe counterexample: `ClientShowNotification`, an FText-bearing Client RPC, is reliably callable from Lua when invoked from a tick on a freshly-resolved controller, and it is the workhorse notify path across every mod I run. So the working rule is: reading these functions in a hook is fine; UpdateChat specifically is proven fatal to call from Lua; any other untested FText RPC is suspect until tried on a dev box.

So local chat interception (a `!command` parser) is pure-Lua territory and always has been. Local chat *delivery* is not. If your feature needs to put a line into someone's chat box, that half has to be C++.

## The C++ deliver recipe

From a C++ side mod, `UpdateChat` works, and it networks to the client cleanly (the call resolves to a Remote callspace, which is what you want — the line shows up on the recipient's machine, not just server-side). The catch is the two string types want opposite handling in the parameter buffer.

```cpp
// UpdateChat(FText Sender, FText Text, FString SenderSteamId,
//            EChatMode ChatMode, bool bIsDev, bool bIsAdmin)
auto deliverChat(UObject* targetController, const StringType& senderLabel,
                 const StringType& body, const StringType& senderSteam, uint8_t mode) -> void
{
    UFunction* fn = FindFn(STR("/Script/TheIsle.TIPlayerController:UpdateChat"));
    if (!fn || !targetController) return;

    FProperty* pSender = fn->FindProperty(FName(STR("Sender"), FNAME_Find));
    FProperty* pText   = fn->FindProperty(FName(STR("Text"), FNAME_Find));
    FProperty* pSteam  = fn->FindProperty(FName(STR("SenderSteamId"), FNAME_Find));
    FProperty* pMode   = fn->FindProperty(FName(STR("ChatMode"), FNAME_Find));
    if (!pSender || !pText || !pSteam || !pMode) return;

    std::vector<uint8_t> buf(fn->GetParmsSize(), 0);
    uint8_t* params = buf.data();

    // FText: safe to build locals and memcpy their bytes into the buffer,
    // because the UE4SS-side FText wrapper has no owning destructor - scope
    // exit frees nothing. Size it by the property (GetSize() reports 16 on
    // this build), NOT by sizeof(your FText wrapper).
    FText senderText{ FString{senderLabel} };
    FText bodyText{ FString{body} };
    std::memcpy(params + pSender->GetOffset_Internal(), &senderText, pSender->GetSize());
    std::memcpy(params + pText->GetOffset_Internal(),   &bodyText,   pText->GetSize());

    // FString: this one OWNS heap. Write it DIRECTLY into the zeroed buffer as the
    // sole owner. A separate owning local plus memcpy double-frees: the local's
    // destructor and the RPC teardown both release the same heap block.
    *std::bit_cast<FString*>(params + pSteam->GetOffset_Internal()) = FString{senderSteam};

    *(params + pMode->GetOffset_Internal()) = mode;

    targetController->ProcessEvent(fn, params);
}
```

The asymmetry is the part worth internalizing, and the reason lives on the wrapper side, not the engine side. The engine's FText is a 16-byte shared-reference handle (a ref-counted pointer to the text data); what makes the local-plus-memcpy pattern safe is that the UE4SS-side FText wrapper carries no owning destructor, so the stack temporary's scope exit releases nothing. The cost is a bounded leak: each constructed FText strands one ref-counted text allocation, so for high-volume delivery build one FText and reuse it. FString owns its buffer outright; if you build an owning local and memcpy it, you now have two owners of one allocation and the second free corrupts the heap on the next RPC. Construct the FString in place, in the buffer, once.

Size by `GetSize()` on the property, not by `sizeof` of whatever FText/FString wrapper your headers give you. On this build `GetSize()` returns 16 for the FText parameter and the property size is authoritative; trusting `sizeof` is how you get a truncated copy that frees garbage.

## Cross-species local delivery

Native local chat is filtered. When a player sends a Spatial line, the engine delivers it to nearby players, but the delivery set is the sender's own species. A nearby player of a different species never receives it. That filter runs where you cannot reach it, so there is no flag to flip and no list to edit.

The approach that works is to leave the native path completely alone and add a parallel delivery for the recipients it skips. Hook `GetChatMessage`, and for each Spatial send, find the nearby players of a different species and call `UpdateChat` on each of those controllers directly. The native filter still serves the same-species recipients, you serve the cross-species ones, and because the two sets are disjoint nobody hears the line twice.

The shape, with the species and proximity checks that keep the delivery sane:

```cpp
// In the GetChatMessage post-hook, queue {senderController, text} for the next
// tick (do not ProcessEvent from inside a hook; defer to on_update). Then:
auto processSpatialSend(UObject* senderCtrl, const StringType& line) -> void
{
    UObject* senderPawn = callGetPawn(senderCtrl);
    if (!senderPawn) return;                       // not spawned
    StringType senderSpecies = shortSpecies(senderPawn);
    double sLoc[3];
    if (!getActorLocation(senderPawn, sLoc)) return;

    for (const StringType& steam : onlineSteams()) {   // presence registry
        UObject* ctrl = resolveControllerBySteam(steam);
        if (!ctrl || !isRealUObject(ctrl)) continue;   // non-null does NOT mean alive; see below
        UObject* pawn = callGetPawn(ctrl);
        if (!pawn) continue;
        StringType species = shortSpecies(pawn);
        if (species == senderSpecies) continue;    // native already served these
        if (!paired(senderSpecies, species)) continue;   // admin matrix, if you use one
        double rLoc[3];
        if (!getActorLocation(pawn, rLoc) || distSq(sLoc, rLoc) > radiusSq) continue;
        deliverChat(ctrl, senderName, line, senderSteam, /*Spatial*/ 0);
    }
}
```

Four things keep this honest. Skip same-species recipients, because the native filter already delivered to them and a second copy is a visible duplicate. Gate on proximity yourself, because `UpdateChat` has no range of its own — it puts the line in whatever controller you hand it, so the radius is your responsibility. Resolve everything live through the presence registry rather than trusting a cached pointer. And validate liveness explicitly on top of that, because live re-resolution alone is not enough: during disconnect teardown a controller lookup can return a non-null pointer to a freed object, and ProcessEvent on it is an uncatchable access violation. Gate every controller behind a UObject-validity check (an `IsReal`-style test against the engine's object array) before any call — and gate the PlayerState behind its own check before reading the sender's display name. The PlayerState dies before its controller during logout teardown, so a controller that passes the validity check can still hand you a freed PlayerState; that exact sequence is the diagnosed cause (from the crash dump) of a real production access violation before the second gate went in.

For player enumeration the loop above uses a presence registry; `GameState.PlayerArray` is the other path and hands you each entry's controller and steam id directly. See the presence registry doc for the trade-off between them. For the species short-name, take the pawn's leaf class `BP_<Species>_C` and strip `BP_` and `_C`.

## On callspace

If you want to confirm the delivery actually networked rather than ran server-local, read the function callspace before the call: `targetController->GetFunctionCallspace(fn, nullptr)`. A Remote bit means the engine is routing it to the owning client. For a Client RPC pushed onto a remote player's controller this is what you expect, and seeing it is a quick sanity check that the line will render on their machine and not just pass through server-side.

## Closing notes

The mental split is the whole thing. The send half of chat (GetChatMessage) is Lua-safe and is where command parsing lives. The deliver half (UpdateChat) crashes from Lua and works from C++, with FText copied in and FString constructed in place. Cross-species local chat is not a filter you unlock; it is a second delivery you add for the recipients the native filter declines to serve, scoped by your own proximity and species checks so the native and added deliveries never overlap.

Global chat needs none of this — it already crosses species on a default server. Spend the effort only if you specifically want proximity-scoped cross-species chat.
