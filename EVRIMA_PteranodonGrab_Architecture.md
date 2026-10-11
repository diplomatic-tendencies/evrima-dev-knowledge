# Pteranodon Grab Mod (The Isle: Evrima)

## Purpose

This document gives the procedure to make a server-side Pteranodon grab mod.

With the mod, a player Pteranodon can grab a live dinosaur below it. The Pteranodon carries the dinosaur in flight and then releases it. The dinosaur can be a player dinosaur or an AI dinosaur. An optional part lets the Pteranodon catch a fish from the water.

The mod changes only the server. Players use the normal game client.

## Application

- Game version: The Isle: Evrima 0.21.784, dedicated server on Windows.
- Tools: a native C++ server mod and a C++ SDK of the game. The tested mod used UE4SS 3.0.1 and an SDK that Dumper-7 made from the server executable.
- [EVRIMA_CPP_Side_Mods.md](EVRIMA_CPP_Side_Mods.md) gives the UE4SS C++ toolchain. This knowledge base does not give the procedure to make the SDK.
- The class names and the function names in this document come from that SDK.

## Contents

- [Terms](#terms)
- [How the method works](#how-the-method-works)
- [1. Mod structure](#1-mod-structure)
- [2. Hooks](#2-hooks)
- [3. Trigger](#3-trigger)
- [4. Checks before a grab](#4-checks-before-a-grab)
- [5. Target search](#5-target-search)
- [6. Grab steps](#6-grab-steps)
- [7. Rules while the carrier holds the target](#7-rules-while-the-carrier-holds-the-target)
- [8. Release](#8-release)
- [9. Logout, destroyed actors, settings change and shutdown](#9-logout-destroyed-actors-settings-change-and-shutdown)
- [10. Player recovery entry (optional)](#10-player-recovery-entry-optional)
- [11. Legs hang point (optional)](#11-legs-hang-point-optional)
- [12. Fish catch (optional)](#12-fish-catch-optional)
- [13. Settings](#13-settings)
- [14. Important notes](#14-important-notes)
- [15. Test status](#15-test-status)

## Terms

| Term | Meaning |
| --- | --- |
| Carrier | The Pteranodon. A player controls it. |
| Target | The dinosaur that the carrier grabs. In the setting names, "victim" means the target. |
| Grab record | The data that the mod keeps for one grab. |
| Trigger input | An ASC server RPC for the right mouse button (section 2.1 and section 3). |
| Release input | A trigger input while the carrier holds a target or a fish item. It releases the target or the fish item. |
| Target event | A hook call that can show a change of the target (section 7.6). |
| Job | A function that the mod puts in a queue. The mod runs it later on the game thread. |
| ASC | The ability system component (`UAbilitySystemComponent`). |
| CMC | The character movement component. |
| Flying CMC | The CMC of the carrier, as a `UTIFlyingDinosaurMovementComponent`. |
| Attach point | The world location of `DragLocation`, or the legs point (section 11). |
| Tested mod | The mod that the tests in section 15 used. |

## How the method works

In the T. rex crush grab, the T. rex (the holder) holds a live dinosaur in its mouth. The mod puts the carrier and its target into the same engine state as this grab.

On the server, the mod sets these values:

- Target: `DraggerCarrier` is the carrier. The ability state is `AS_Crushed` (53).
- Carrier: the ability state is `AS_CrushPoseStarted` (5). The T. rex grab uses this state for the holder.
- The mod attaches the target to the `DragLocation` component of the carrier.

Each `ATICharacterBase` has a `DragLocation` scene component. The game attaches this component to the mesh socket `PickUpSocket`. On a Pteranodon, this point is at the mouth.

On a client, `ATICharacterBase::OnRep_AbilityStateChanged` runs when the ability state changes. For state 53 with a `DraggerCarrier`, it stops the movement of the character. Then it attaches the character to `DraggerCarrier->DragLocation`. This code does not check the species of the carrier.

> The client behavior above comes from a static analysis of the server executable. The server executable contains this client code. The examination included only the server executable.

## 1. Mod structure

### 1.1 Hooks and jobs

1. Use a hook on `UObject::ProcessEvent` to monitor the engine calls. With UE4SS, use `RC::Unreal::Hook::RegisterProcessEventPreCallback` and `RegisterProcessEventPostCallback`. The callback gets the object, the `UFunction` and the parameter block.
2. Install the callbacks after the server starts, not during the early boot. [EVRIMA_SpeciesMixpack_Architecture.md](EVRIMA_SpeciesMixpack_Architecture.md) describes a server crash from an install during the early boot.
3. Find each `UFunction` pointer from section 2 one time. For example, use `SDK::UClass::GetFunction(className, functionName)`. Keep the pointers in a list.
4. In the pre-callback, compare the `UFunction` pointer with your list. Copy the necessary parameters. Then put a job in the queue.
5. Do not change game objects in the pre-callback. Do all engine reads and writes in jobs.
6. Run the jobs on the game thread. The tested mod runs them in the post-callback when these conditions are true:
   - The callback runs on the game thread.
   - The `ProcessEvent` call is not inside another `ProcessEvent` call.
   - No network function is on the `ProcessEvent` stack. A network function has a `FUNC_Net*` flag.
7. Set a maximum time for the jobs in one post-callback. Keep the other jobs for the next post-callback.

### 1.2 Events only

Use engine events and one-shot timers. Do not poll the game each frame. The tested mod has no per-frame work.

A worker thread can wait for the timers. At the end time of a timer, the worker puts the timer job in the game-thread queue. The mod uses timers for these tasks, for example:

- The maximum hold time.
- The end of each grace time (section 7.3).
- The state reset after a release (section 8.3).
- The legs hang point (section 11).

### 1.3 Object identity

The game can destroy a carried pawn between two events. Thus keep a safe identity for each stored object:

- The pointer.
- The index of the object in the global object array.
- The `FName` of the object (`ComparisonIndex` and `Number`).

Before the mod uses a stored object, do a check of the array index. It must still hold the same pointer and the same name. For an actor, also make sure that `bActorIsBeingDestroyed` is false.

### 1.4 Setters and headers

- Use the engine setters, for example `SetDraggerCarrier` and `SetAbilityState`. Do not write these fields directly. The setters tell the replication system that the properties changed (push model).
- Write a field directly only when it has no setter.
- Keep the UE4SS headers and the SDK headers in different source files. The two header sets declare types with the same names. Send raw pointers between the files.

## 2. Hooks

### 2.1 Trigger hooks

| Function | Object of the call | Parameters | Purpose |
| --- | --- | --- | --- |
| `UTIFlyingDinosaurMovementComponent::ServerSetIsAirBraking` | Flying CMC | `bNewIsAirBraking` | Air brake on or off |
| `UAbilitySystemComponent::ServerTryActivateAbility` | ASC | `AbilityToActivate.Handle`, `InputPressed` | Trigger input |
| `UAbilitySystemComponent::ServerTryActivateAbilityWithEventData` | ASC | `AbilityToActivate.Handle`, `InputPressed` | Trigger input |
| `UAbilitySystemComponent::ServerSetInputPressed` | ASC | `AbilityHandle.Handle` | Trigger input |
| `UAbilitySystemComponent::ServerAbilityRPCBatch` | ASC | `BatchInfo.AbilitySpecHandle.Handle`, `BatchInfo.InputPressed` | Trigger input when `InputPressed` is true |
| `UAbilitySystemComponent::ServerSetInputReleased` | ASC | `AbilityHandle.Handle` | Not a trigger input. Check event only (section 7.1). |
| `UAbilitySystemComponent::ServerSetReplicatedEvent` | ASC | `AbilityHandle.Handle`, `EventType` | Not a trigger input. Check event only (section 7.1). |

### 2.2 Safety hooks

| Function | Object of the call | Parameters | Purpose |
| --- | --- | --- | --- |
| `BP_SurvivalGameMode_C::K2_OnLogout` | Game mode | `ExitingController` | A player left |
| `ATIPlayerController::PrepareSafeLogout` | Player controller | None | A safe logout started |
| `UAbilitySystemComponent::OnOwnerActorDestroyed` | ASC | `InActor` | The game destroys the carrier or the target |
| `UAbilitySystemComponent::OnAvatarActorDestroyed` | ASC | `InActor` | The game destroys the carrier or the target |

Find the `UFunction` pointers of all safety hooks before the first grab. Reject each grab until the mod has all these pointers.

### 2.3 Release hooks

| Function | Object of the call | Parameters | Purpose |
| --- | --- | --- | --- |
| `UTIFootstepsComponent::OnCharacterMovementChanged` | Footsteps component | `Character`, `PrevMovementMode` | Movement mode of the carrier or the target changed (the footsteps hook) |
| `ATIFlyingDinosaurBase::OnCapsuleHit` | Carrier | `OtherActor`, `Hit` | The carrier hit an object |
| `UTICharacterMovementComponentBase::OnBeginOverlap` | CMC of the carrier | `OtherActor` | The carrier touched an actor |
| `ATICharacterBase::ReleasePhysicsCharacter` | The character that died | None | Death of the carrier or the target (the death hook) |
| `BP_SurvivalGameMode_C::OnACharacterDied` | Game mode | None | A character died: do the checks for all grabs again |
| `UTIFlyingDinosaurMovementComponent::ServerSetWaterCrashed` | Flying CMC | `bNewIsWaterCrashed` | The carrier crashed into water |
| `UTIFlyingDinosaurMovementComponent::ServerSetIsCrashed` | Flying CMC | `bNewIsCrashed` | The carrier crashed |
| `ATICharacterBase::ServerRequestToGetUp` | Target | None | Target event |
| `UTICharacterMovementComponentBase::StaggerClient` | CMC of the target | None | Target event |
| `UTICharacterMovementComponentBase::KnockdownClient` | CMC of the target | None | Target event |

The footsteps hook and the death hook are necessary. Reject each grab until the mod has the pointers of these two hooks.

### 2.4 Recovery hook (optional)

| Function | Object of the call | Parameters | Purpose |
| --- | --- | --- | --- |
| `APlayerController::ClientRestart` | Player controller | `NewPawn` | Move a recovered player to the ground (section 10) |

### 2.5 Hook notes

- A hook on `UObject::ProcessEvent` does not see an empty Blueprint event on an actor. `AActor::ProcessEvent` returns early when the function is not native and has no script.
- Thus the hook receives no `K2_OnMovementModeChanged`, `OnLanded` or `ReceiveHit` calls. The static analysis covers all three events. In the tests, the hook received no `K2_OnMovementModeChanged` call.
- The hook receives a Blueprint event that has a Blueprint script. `BP_SurvivalGameMode_C::K2_OnLogout` is an example.
- `UTIFootstepsComponent::OnCharacterMovementChanged` is native. The character calls it through `MovementModeChangedDelegate` after each movement mode change. Use it to find when the carrier lands, goes into the water or crashes.
- The hook receives the server RPCs from a client. In the tests, one right-click in flight sent two `ServerTryActivateAbility` calls. One call was for the ability `GA_Skim_C`, and one call was for `UTIGameplayAbilityTryLatch`. Both calls had the input ID 2. No `ServerAbilityRPCBatch` call occurred.
- The ASC hooks receive each ability call on the server. Thus keep two short lists of ASCs:
  - Carrier list: put the ASC of a carrier in the list at each brake event. Keep it in the list while the brake is on, and for `brake_window_ms` after the brake changes to off. Also keep it in the list while the carrier holds a target, or a fish item with `fish_rmb_drop` 1.
  - Target list: put the ASC of each target in the list at the grab. Remove it at the release.
  - In the pre-callback, use an ASC call only when its ASC is in one of the two lists. Discard all other ASC calls.

## 3. Trigger

The player flies, holds the air brake key and pushes the right mouse button. In the tests, the air brake key was Z.

At a brake event (`ServerSetIsAirBraking`), do these steps:

1. Find the owner of the flying CMC.
2. Continue only for a live `ATIPteranodon` with an `ATIPlayerController`.
3. Record the brake state (on or off) and the time for the ASC of this carrier.

At a trigger input, do these steps:

1. Find the owner of the ASC. Continue only for a live `ATIPteranodon` with an `ATIPlayerController`.
2. Find the ability spec in `ASC->ActivatableAbilities.Items` with the same handle. Continue only when the ability of the spec is a `UTIGameplayAbilityTryLatch` or a `UTIGameplayAbilitySkim`.
3. If the carrier holds a target, apply the release rule (below). Then stop.
4. If the carrier holds a fish item and `fish_rmb_drop` is 1, apply the input rules (below). If the input rules accept the input, do the fish item release (section 12.6). Then stop.
5. Make sure that these conditions are true. If one condition is false, stop.
   - `trigger_mode` is 1.
   - `IsFlying()` of the carrier is true.
   - `GetIsCrashed()` and `GetIsCrashLanded()` of the flying CMC are false.
   - The air brake is on now (`IsAirBraking()`), or it changed 400 ms or less before the trigger input.
6. Apply the input rules (below). If the input rules accept the input, put the grab job in the queue.

The release rule:

- Release the target when `release_input` is 1 and the grab started `press_dedupe_ms` (300 ms) or more before the input. Then record the input as an accepted input.
- If not, discard the input.

The input rules:

- Discard an input that comes less than `press_dedupe_ms` (300 ms) after the last input of this carrier. An input that this rule discards does not start a new 300 ms period.
- Discard an input that comes less than `cooldown_ms` (1000 ms) after the last accepted input of this carrier. An input that this rule discards starts a new 300 ms period.
- Accept all other inputs.

On the ground, the same key also sends `ServerSetIsAirBraking(true)`. Thus always check `IsFlying()`.

```cpp
bool IsGrabInput(SDK::UAbilitySystemComponent* Asc, int32_t Handle)
{
    const int32_t Count = Asc->ActivatableAbilities.Items.Num();
    for (int32_t i = 0; i < Count; ++i)
    {
        const SDK::FGameplayAbilitySpec& Spec = Asc->ActivatableAbilities.Items[i];
        if (Spec.Handle.Handle != Handle || !Spec.Ability)
            continue;
        return Spec.Ability->IsA(SDK::UTIGameplayAbilityTryLatch::StaticClass())
            || Spec.Ability->IsA(SDK::UTIGameplayAbilitySkim::StaticClass());
    }
    return false;
}
```

## 4. Checks before a grab

Do these checks at the start of the grab job. If one check fails, do not grab.

1. `enabled` is 1. The mod has the pointers of the safety hooks, the footsteps hook and the death hook (section 2).
2. The number of active grabs is less than `max_active`.
3. The carrier is a live `ATIPteranodon` (`GetIsAlive()`). An `ATIPlayerController` controls it.
4. The carrier is not in a grab record. `IsFlying()` of the carrier is true.
5. `GetIsCrashed()` and `GetIsCrashLanded()` of the flying CMC are false.
6. The carrier holds nothing: `GetDraggedPickablePiece()` and `GetDraggedActor()` are null.
7. `DragLocation` of the carrier is a scene component. Its owner is the carrier. Its attach socket name is `PickUpSocket`.
8. The root component of the carrier is a `UCapsuleComponent`. Its collision object type is `ECC_Pawn`.
9. The footsteps event works for this carrier:
   - `FootstepsComponent` of the carrier is set.
   - `BeginPlay` is complete for this component. The SDK does not name this bit (`bHasBegunPlay`). On game version 0.21.784, it is byte 0x9F, mask 0x20 of the component.
   - `MovementModeChangedDelegate.InvocationList` of the carrier has an entry for this component and the function `OnCharacterMovementChanged`. In each entry, compare `Object.ObjectIndex` with the index of the component. Compare `FunctionName` with the name of the function.

Check 9 is necessary. The rule "carrier landed" needs this event.

## 5. Target search

### 5.1 Sweep

Use these values:

| Value | Meaning |
| --- | --- |
| P | The carrier location (`K2_GetActorLocation()`) |
| HH | The scaled half height of the carrier capsule |
| FeetZ | P.Z - HH |
| V | The carrier velocity (`GetVelocity()`) |
| R | `radius_cm` (default 120) |
| MaxGap | The smaller value of `reach_cm` (default 300) and 600 |

Do these steps:

1. Calculate the lead L = (V.X, V.Y) multiplied by `lead_s` (default 0.10). If L is longer than `lead_max_cm` (default 150), make the length of L equal to `lead_max_cm`.
2. Sweep a sphere down with `UKismetSystemLibrary::SphereTraceSingleForObjects`. Use these values:
   - Start: (P.X + L.X, P.Y + L.Y, FeetZ - R - 2).
   - End: the same X and Y. End Z = start Z - (MaxGap + 2R).
   - Radius: R.
   - Object type: `ObjectTypeQuery3` (the Pawn object type).
   - Actors to ignore: the carrier, and each actor that failed before. The list has a maximum of 5 actors.
3. Find the actor from the hit. Use `HitObjectHandle.ReferenceObject`. If it is a component, use its owner. If it is null, use the owner of `Component`.
4. Stop the search when the sweep has no hit, the hit has no actor, or the actor is the carrier.
5. Apply the target filter (section 5.2), the geometry check (section 5.3) and the line-of-sight check (section 5.4).
6. If the actor fails, add it to the list of actors to ignore. Then sweep again. Do a maximum of `max_sweeps` sweeps (default 4).

```cpp
SDK::AActor* SweepOnce(SDK::ATICharacterBase* Carrier, const SDK::FVector& Start, const SDK::FVector& End,
                       float Radius, SDK::AActor** Ignore, int32_t IgnoreCount)
{
    SDK::EObjectTypeQuery Types[1] = { SDK::EObjectTypeQuery::ObjectTypeQuery3 };
    SDK::TArray<SDK::EObjectTypeQuery> TypeArray(Types, 1, 1);
    SDK::TArray<SDK::AActor*> IgnoreArray(Ignore, IgnoreCount, 5);
    SDK::FHitResult Hit{};
    if (!SDK::UKismetSystemLibrary::SphereTraceSingleForObjects(Carrier, Start, End, Radius, TypeArray, false, IgnoreArray,
                                                                SDK::EDrawDebugTrace::None, &Hit, true,
                                                                SDK::FLinearColor{}, SDK::FLinearColor{}, 0.0f))
        return nullptr;
    SDK::UObject* Ref = Hit.HitObjectHandle.ReferenceObject.Get();
    SDK::UObject* Owner = nullptr;
    if (Ref && Ref->IsA(SDK::AActor::StaticClass()))
        Owner = Ref;
    else if (Ref && Ref->IsA(SDK::UActorComponent::StaticClass()))
        Owner = static_cast<SDK::UActorComponent*>(Ref)->GetOwner();
    if (!Owner && Hit.Component.Get())
        Owner = Hit.Component.Get()->GetOwner();
    if (!Owner || Owner == Carrier || !Owner->IsA(SDK::AActor::StaticClass()))
        return nullptr;
    return static_cast<SDK::AActor*>(Owner);
}
```

The function returns null when the search must stop. The caller applies the target filter to the actor. The data of the SDK `TArray` in this example is in stack memory. The engine reads it and does not free it.

### 5.2 Target filter

Reject the actor if one of these conditions is true. The tested mod checks them in this sequence.

| Check | Reject the actor when |
| --- | --- |
| Self | The actor is the carrier. |
| Class | The actor is not an `ATIDinosaurBase`, or the actor is an `ATIFishBase`. |
| Life | `GetIsAlive()` is false, or `bActorIsBeingDestroyed` is true. |
| Controller | The controller is not an `ATIPlayerController`. When `allow_ai_target` is 1, also accept an AI controller. An AI controller is a controller that is not an `APlayerController`. A pawn with no controller is not accepted. |
| Ability state | The state is in the safety list below. When `target_require_as84` is 1, reject all states except `AS_None` (84). |
| Carry | `GetDraggerCarrier()` or `GetDraggedActor()` is not null, or the actor is in a grab record. |
| Flight | `IsFlying()` is true, and `allow_flying_target` is 0. |
| Water | `IsSwimming()` is true, and `allow_swimming_target` is 0. |
| Pounce | `IsPounced()` or `IsGrappled()` is true. |
| Weight | `weight_ratio` is more than 0, and the target `GetWeight()` is 0 or less. Or: `weight_ratio` is more than 0, and the carrier `GetWeight()` is less than `weight_ratio` multiplied by the target `GetWeight()`. |

The safety list of ability states:

| Value | `EAbilityState` |
| --- | --- |
| 0, 1 | `AS_DraggingCorpseL`, `AS_DraggingCorpseR` |
| 2 | `AS_HoldingObject` |
| 3, 4, 5 | `AS_LungeHolding`, `AS_CrushHolding`, `AS_CrushPoseStarted` |
| 10, 11 | `AS_PounceAttackingSmall`, `AS_PounceAttackingBig` |
| 18 to 21 | `AS_PouncedHitSmall`, `AS_PouncedHitBig`, `AS_PouncedStruggleSmall`, `AS_PouncedStruggleBig` |
| 22, 23 | `AS_Grappled`, `AS_GrappledFront` |
| 27 to 30 | `AS_Swallowed`, `AS_Carried`, `AS_Dragged`, `AS_Corpse` |
| 52, 53 | `AS_Lunged`, `AS_Crushed` |
| 59, 60 | `AS_Embryo`, `AS_Hatching` |
| 69, 70 | `AS_PouncedHitSmallF`, `AS_PouncedStruggleSmallF` |

### 5.3 Geometry check

Use these values: TZ is the target location Z. THH and TR are the scaled half height and the scaled radius of the target capsule.

1. Calculate the vertical gap G = FeetZ - (TZ + THH). Reject the target when G is less than -THH or more than MaxGap.
2. Calculate the horizontal distance from (P.X + L.X, P.Y + L.Y) to the target. Reject the target when the distance is more than R + TR.

### 5.4 Line of sight

Do this check only when `los_check` is 1.

1. Trace a line with `UKismetSystemLibrary::LineTraceSingle` and `TraceTypeQuery1` (Visibility):
   - Start: (P.X, P.Y, FeetZ).
   - End: (target X, target Y, TZ + THH / 2).
   - Actors to ignore: the carrier and the target.
2. If no blocking hit occurs, the line is clear.
3. If the owner of the hit component is water, add it to the list of actors to ignore. Then trace again. Water is an `ATIWater` actor, or an actor with `AquaticSurface` or `TIWater` in its class name.
4. Do a maximum of 4 traces. If the fourth trace also hits water, reject the target.
5. Reject the target for each other blocking hit.

Rivers are `ATIWater` actors, for example `BP_FlowmapRiverSystem_C`. Without the water rule, the line trace stops at the water surface. Then the mod cannot grab a target in the water.

## 6. Grab steps

First make the grab record. Copy the current settings into the record. The grab uses this copy until the release. Exceptions: `enabled`, `release_input` and `press_dedupe_ms` come from the current settings.

With `attach_delay_ms` 0 (default), do all steps in one job. With a larger value, do step 10 in a timer job after this delay. Before step 10, make sure again that the grab conditions are true.

Before step 1, make sure that these conditions are true:

- The target CMC (`CharacterMovementComponent`) is a `UTICharacterMovementComponentBase`.
- The target root component is a `UCapsuleComponent`.
- The target has an ASC.
- `SDK::UGE_GrantLungedTag_C::StaticClass()` returns the class. If it returns null, reject the grab.
- The carrier has a `DragLocation` component.

Do the steps in this sequence. After each step, read the value back. If a step fails, release the target (section 8). Do only the release steps of the completed grab steps.

| Step | Call | Read back |
| --- | --- | --- |
| 1 | Target CMC: `DisableMovement()`, `StopMovementImmediately()`, `SetRequestedSpeed({0, 0, 0})` | None |
| 2 | `Target->SetDraggerCarrier(Carrier)` | `GetDraggerCarrier()` is the carrier |
| 3 | Keep the old response of the target capsule to `ECC_Pawn`. Then set it to `ECR_Ignore`. | The response is `ECR_Ignore` |
| 4 | Target ASC: `BP_ApplyGameplayEffectToSelf(GE_GrantLungedTag_C, 1.0, empty context)`. Keep the handle. | The handle is not -1, and `GetGameplayEffectCount` is 1 or more |
| 5 | `Target->CancelAbilities(Tags)` with the 17 tags below | None |
| 6 | `Carrier->SetAbilityState(AS_CrushPoseStarted)`, or the state in `carrier_state` | The state is 5 (`carrier_state`) |
| 7 | `Target->SetAbilityState(AS_Crushed)` | The state is 53 |
| 8 | Target CMC: keep the old value of `bIgnoreClientMovementErrorChecksAndCorrection`. Then set it to 1. | The value is 1 |
| 9 | Target CMC: keep the old value of `bKnockdownImmune`. Then set it to 1. | The value is 1 |
| 10 | `Target->K2_AttachToComponent(Carrier->DragLocation, NAME_None, SnapToTarget, SnapToTarget, KeepWorld, false)` | `GetAttachParentActor()` is the carrier |

After step 10, the grab is active. Then do these steps:

- Start the maximum hold timer (`max_hold_ms`).
- Start the grace timers (section 7.3).
- Start the legs timer when the legs hang point is on (section 11).
- Write the recovery entry for a player target (section 10).

The 17 tags for step 5:

```text
Ability.Skill.Nest
Ability.Skill.Rest
Ability.Skill.Wallow
Ability.Skill.Beg
Ability.Skill.Scent
Ability.Skill.PrimaryAttack
Ability.Skill.Vocalize
Ability.Skill.VoiceBroadcast
Ability.Skill.VoiceFriendly
Ability.Skill.VoiceThreaten
Ability.Skill.VoiceHelp
Ability.Skill.SecondaryAttackStationary
Ability.Skill.SecondaryAttack.UtahPounce
Ability.Skill.SecondaryAttack.HeadButt
Ability.Skill.SecondaryAttack.DeinoLunge
Ability.Skill.SecondaryAttack.TroodonPounce
Status.Launching
```

The asset path of the gameplay effect class:

```text
/Game/TheIsle/Core/Characters/Dinosaurs/Abilities/Effects/GE_GrantLungedTag.GE_GrantLungedTag_C
```

On game version 0.21.784, the two movement bits have these locations:

| Bit | Class | Byte offset | Mask |
| --- | --- | --- | --- |
| `bIgnoreClientMovementErrorChecksAndCorrection` | `UCharacterMovementComponent` | 0x565 | 0x02 |
| `bKnockdownImmune` | `UTICharacterMovementComponentBase` | 0x1004 | 0x40 |

The tested mod finds both bits by name through reflection (`FBoolProperty`). When the mod starts, it compares them with the values above. If a value is different, it rejects all grabs.

```cpp
enum GrabStep : uint32_t
{
    StepMove = 1u << 0,
    StepCarrier = 1u << 1,
    StepCapsule = 1u << 2,
    StepEffect = 1u << 3,
    StepCarrierState = 1u << 4,
    StepTargetState = 1u << 5,
    StepCorrection = 1u << 6,
    StepKnockdown = 1u << 7,
    StepAttach = 1u << 8,
};

struct GrabRecord
{
    uint32_t Done = 0;
    SDK::ECollisionResponse OldPawnResponse = SDK::ECollisionResponse::ECR_Overlap;
    bool OldIgnoreCorrection = false;
    bool OldKnockdownImmune = false;
    SDK::FActiveGameplayEffectHandle Effect{};
};

bool GrabSteps(GrabRecord& Rec, SDK::ATICharacterBase* Carrier, SDK::ATICharacterBase* Target,
               SDK::UClass* LungedEffect, SDK::FGameplayTag* Tags, int32_t TagCount)
{
    SDK::UTICharacterMovementComponentBase* Cmc = Target->CharacterMovementComponent;
    auto* Capsule = static_cast<SDK::UCapsuleComponent*>(Target->K2_GetRootComponent());
    SDK::UAbilitySystemComponent* Asc = SDK::UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(Target);

    Cmc->DisableMovement();
    Rec.Done |= StepMove;
    Cmc->StopMovementImmediately();
    Cmc->SetRequestedSpeed(SDK::FVector{ 0.0, 0.0, 0.0 });

    Target->SetDraggerCarrier(Carrier);
    Rec.Done |= StepCarrier;
    if (Target->GetDraggerCarrier() != Carrier)
        return false;

    Rec.OldPawnResponse = Capsule->GetCollisionResponseToChannel(SDK::ECollisionChannel::ECC_Pawn);
    Capsule->SetCollisionResponseToChannel(SDK::ECollisionChannel::ECC_Pawn, SDK::ECollisionResponse::ECR_Ignore);
    Rec.Done |= StepCapsule;
    if (Capsule->GetCollisionResponseToChannel(SDK::ECollisionChannel::ECC_Pawn) != SDK::ECollisionResponse::ECR_Ignore)
        return false;

    SDK::FGameplayEffectContextHandle EmptyContext{};
    Rec.Effect = Asc->BP_ApplyGameplayEffectToSelf(SDK::TSubclassOf<SDK::UGameplayEffect>(LungedEffect), 1.0f, EmptyContext);
    Rec.Done |= StepEffect;
    if (Rec.Effect.Handle == -1
        || Asc->GetGameplayEffectCount(SDK::TSubclassOf<SDK::UGameplayEffect>(LungedEffect), nullptr, false) < 1)
        return false;

    SDK::FGameplayTagContainer Container{};
    Container.GameplayTags = SDK::TArray<SDK::FGameplayTag>(Tags, TagCount, TagCount);
    Target->CancelAbilities(Container);

    Carrier->SetAbilityState(SDK::EAbilityState::AS_CrushPoseStarted);
    Rec.Done |= StepCarrierState;
    if (Carrier->GetAbilityState() != SDK::EAbilityState::AS_CrushPoseStarted)
        return false;

    Target->SetAbilityState(SDK::EAbilityState::AS_Crushed);
    Rec.Done |= StepTargetState;
    if (Target->GetAbilityState() != SDK::EAbilityState::AS_Crushed)
        return false;

    Rec.OldIgnoreCorrection = Cmc->bIgnoreClientMovementErrorChecksAndCorrection;
    Cmc->bIgnoreClientMovementErrorChecksAndCorrection = 1;
    Rec.Done |= StepCorrection;
    if (Cmc->bIgnoreClientMovementErrorChecksAndCorrection != 1)
        return false;

    Rec.OldKnockdownImmune = Cmc->bKnockdownImmune;
    Cmc->bKnockdownImmune = 1;
    Rec.Done |= StepKnockdown;
    if (Cmc->bKnockdownImmune != 1)
        return false;

    const bool Attached = Target->K2_AttachToComponent(Carrier->DragLocation, SDK::FName{},
                                                       SDK::EAttachmentRule::SnapToTarget, SDK::EAttachmentRule::SnapToTarget,
                                                       SDK::EAttachmentRule::KeepWorld, false);
    Rec.Done |= StepAttach;
    return Attached && Target->GetAttachParentActor() == Carrier;
}
```

Make each tag one time: set `FGameplayTag::TagName` to `UKismetStringLibrary::Conv_StringToName(FString(L"<tag>"))`.

Notes:

- Steps 1 to 8 and step 10 copy the T. rex grab in the server executable. The T. rex grab also sets `DraggerCarrier` before the state 53.
- In step 4, the T. rex grab applies the same effect class from the holder. The tested mod applies it with the target ASC.
- The T. rex grab does not set `bKnockdownImmune`. The tested mod adds step 9.
- The T. rex grab also changes the turn rate of the holder and starts a crush damage timer. The tested mod does not do these steps.
- The T. rex grab also sets `DraggedActor` of the holder. By default, the tested mod does not set it (`set_dragged_actor` 0).
- Use an empty `FGameplayEffectContextHandle` in step 4. The engine function makes its own context.

## 7. Rules while the carrier holds the target

### 7.1 Events

Each event below puts a check job in the queue. The check job does the checks in section 7.2. Some events also have their own release rule.

| Event | Own rule |
| --- | --- |
| Footsteps event of the carrier | None. Do the checks in section 7.2. |
| `ServerSetIsCrashed(true)` of the carrier | None. Do the checks in section 7.2. |
| `ServerSetWaterCrashed(true)` of the carrier | Release with "carrier water" when `drop_on_water` is 1, after the land grace time (section 7.3). The tests did not include this rule. |
| `OnCapsuleHit` of the carrier | See section 7.4. |
| `OnBeginOverlap` of the carrier CMC | See section 7.5. |
| A target event: footsteps event of the target, `ServerRequestToGetUp`, `StaggerClient`, `KnockdownClient`, or an ASC call of the target | See section 7.6. |
| `ReleasePhysicsCharacter` of the carrier or the target | None. Do the checks in section 7.2. |
| `OnACharacterDied` | Do the checks in section 7.2 for all grabs. |
| `OnOwnerActorDestroyed` or `OnAvatarActorDestroyed` with the carrier or the target | Release with "carrier gone" or "target gone". |
| Logout or safe logout of the carrier player or the target player | Release (section 9). |
| Brake event of the carrier | None. Do the checks in section 7.2. |
| ASC call of the carrier | Do the checks in section 7.2. A trigger input can also release the target (section 3). |
| Maximum hold timer | Do the checks in section 7.2. Then release with "timeout". |
| Grace timers | Do the checks in section 7.2 and the target hit check (section 7.6). |

The hit hook discards a carrier hit during the grace time, and when `drop_on_hit` and `drop_on_land` are both 0. For a hit, the grace time is the shorter of `land_grace_ms` and `hit_grace_ms` when both rules are on. The touch hook discards a carrier touch during `hit_grace_ms`, and when `drop_on_touch` is 0.

Your own jobs also change the target, for example in the grab steps. Do not use the target events that your own jobs cause. The tested mod sets a thread-local flag while its grab, release, legs, nudge and fish jobs run. The footsteps hook discards the target events while this flag is set.

### 7.2 Checks

Do the checks in this sequence. Release the target at the first rule that applies.

| Rule | Release when |
| --- | --- |
| Disabled | `enabled` is 0. |
| Carrier gone | The carrier identity is not valid (section 1.3). |
| Target gone | The target identity is not valid. |
| Grab not complete | Grab step 10 is not complete after `attach_delay_ms` + 1000 ms. |
| Target dead | `bIsDead` is true, or the target state is `AS_Corpse` (30). |
| Carrier dead | `bIsDead` is true, or the carrier state is `AS_Corpse` (30). |
| Carrier lost | `GetDraggerCarrier()` of the target is not the carrier. |
| Target left 53 | The target state is not 53. |
| Carrier left carrier state | The carrier state is not `carrier_state` (default 5). |
| Detached | `GetAttachParentActor()` of the target is not the carrier. |
| Legs parent | Legs hang point only: the target root is not attached to the legs parent. |
| Distance | The target is more than `max_dist_cm` from the attach point. |
| Target hit | The target hit check (section 7.6) found a hit after the target hit grace time. |
| Carrier landed | The carrier movement mode is `MOVE_Walking` or `MOVE_NavWalking`. |
| Carrier water | The carrier movement mode is `MOVE_Swimming`. |
| Carrier mode | The carrier movement mode is `MOVE_None` or `MOVE_Custom`. |
| Carrier crashed | `GetIsCrashed()` of the flying CMC is true. |
| Controller | The controller of the carrier or the target changed. |
| Timeout | The hold time is more than `max_hold_ms`. |

The carrier rules have these settings:

- "Carrier landed" and "carrier mode" apply only when `drop_on_land` is 1, after the land grace time.
- "Carrier water" applies only when `drop_on_water` is 1, after the land grace time.
- "Carrier crashed" applies only when `drop_on_hit` is 1, after the hit grace time.
- The tested mod checks "carrier crashed" only when the carrier is in `MOVE_Falling` or `MOVE_Flying`.

Do the death test with `bIsDead` or with the state `AS_Corpse`. `OnACharacterDied` can run before the game sets these values. Thus use `ReleasePhysicsCharacter` as the main death event.

### 7.3 Grace times

The grace times start when grab step 10 is complete. A rule does not apply during its grace time.

| Setting | Default | Name | Rules |
| --- | --- | --- | --- |
| `land_grace_ms` | 500 | Land grace time | Carrier landed, carrier water, carrier mode |
| `hit_grace_ms` | 500 | Hit grace time | Carrier crashed, carrier hit, carrier touch |
| `victim_hit_grace_ms` | 1500 | Target hit grace time | Target hit |

An event can come during a grace time. It is possible that no other event comes after it. Thus start a timer for the end of each grace time. At the end time, do the checks again.

The tested mod starts the first timer at the larger of `land_grace_ms` and `hit_grace_ms`, plus 50 ms. If `drop_on_victim_hit` is 1 and `victim_hit_grace_ms` is larger, it starts a second timer at `victim_hit_grace_ms` plus 50 ms.

### 7.4 Carrier hit

Use `OnCapsuleHit` of the carrier.

1. Do not use the hit when `OtherActor` is the target.
2. Calculate D = `ImpactPoint.Z` - (`Location.Z` - carrier half height). If `ImpactPoint` is (0, 0, 0), use `Location.Z` for the impact Z.
3. Continue with step 4 only when the carrier is in flight. In flight means `MOVE_Falling` or `MOVE_Flying`. If not, go to step 6.
4. If D is not more than half of the carrier capsule radius, the carrier landed. Release with "carrier landed" when `drop_on_land` is 1, after the land grace time.
5. If D is more than half of the carrier capsule radius, the carrier hit a wall. Release with "carrier hit" when `drop_on_hit` is 1, after the hit grace time.
6. If no release occurs, do the target hit check (section 7.6).

### 7.5 Carrier touch

Use `OnBeginOverlap` of the carrier CMC. Release with "carrier touch" when all these conditions are true:

- `drop_on_touch` is 1.
- The touch occurs after the hit grace time.
- The carrier is in flight (`MOVE_Falling` or `MOVE_Flying`).
- `OtherActor` is a live `ATICharacterBase`. It is not the carrier, and it is not the target.

### 7.6 Target events and target hit

At a target event, do these steps:

1. If the target is dead, release with "target dead".
2. Release the target when one of these conditions is true:
   - The target movement mode is not `MOVE_None`.
   - The target state is not 53.
   - The target is not attached to the carrier.
3. If no condition is true, do the target hit check.

The target hit check runs a maximum of one time in 250 ms. It applies only when `drop_on_victim_hit` is 1, after the target hit grace time.

1. Sweep the target shape at its current location, with a length of 1 cm in Z. Use `ObjectTypeQuery1` and `ObjectTypeQuery2` (WorldStatic and WorldDynamic). The actors to ignore are the carrier and the target.
2. The shape is a sphere with the capsule radius. For an upright target, the shape is the capsule. An upright target hangs at the legs point, and the absolute values of `legs_pitch` and `legs_roll` are less than 1.
3. The mod accepts a hit when the hit component blocks `ECC_Pawn`. If it does not block, add its owner to the list of actors to ignore. Then sweep again. Do a maximum of 4 sweeps.
4. If the mod accepts no sweep hit, find the ground below the target (section 8.1, step 3). Start the trace at the higher Z of the top of the target shape and the carrier.
5. The lowest point of the target is its Z minus the capsule radius. For an upright target, use the half height. The mod accepts a hit when this point is less than 5 cm above the ground. Water is not ground for this check.
6. When the mod accepts a hit, it records a safe point. For a sweep hit, the safe point is the hit location. A sweep that started inside the object gives no safe point. For a ground hit, the safe point is the target location with its lowest point 10 cm above the ground.

## 8. Release

### 8.1 Measurements

Do these measurements before release step 2. Do them only when grab step 10 is complete.

1. Find the target location and the capsule size.
2. Do a test to find if the target is inside geometry. Use `UKismetSystemLibrary::CapsuleTraceSingle` from the target location to 1 cm above it, with `TraceTypeQuery1`. The target is inside when `bBlockingHit` and `bStartPenetrating` are both true.
3. Find the ground below the target. Use `LineTraceSingle` with `TraceTypeQuery1`, 50000 cm down. The actors to ignore are the carrier and the target.
   - Start Z: the top of the target capsule. If the horizontal distance from the carrier to the target is 300 cm or less, start at the carrier Z.
   - If the class name of the hit owner contains `InstancedFoliageActor`, add this owner to the list of actors to ignore. Then trace again. Do a maximum of 4 traces.
   - A hit point near (0, 0, 0) is not a ground hit. Near means less than 1 cm on each axis.
   - Record if the hit actor is an `ATIWater`.
4. If the trace finds no ground, use the safe point (section 7.6, step 6). Do this only for the release reason "target hit".
5. The target is buried when these two conditions are true:
   - The ground is not water.
   - The ground Z is more than 1 cm above the target Z minus the capsule half height.

### 8.2 Placement rule

Put the target on the ground when a ground point exists and one of these conditions is true:

- The target is buried, or the safe point is in use.
- The target is inside geometry.
- `drop_mode` is 1.
- The carrier did not cause the release.

The carrier causes these releases:

- The release input.
- "Carrier landed", "carrier water", "carrier mode", "carrier crashed", "carrier hit" and "carrier touch" (sections 7.2, 7.4 and 7.5).
- "Target hit" (section 7.6).

A safe logout of the target player uses the same placement rule as a release that the carrier caused.

All other releases put the target on the ground. Examples: timeout, carrier death, logout, settings change and admin command.

After a release that the carrier caused, the target falls from the release point (`drop_mode` 0). Set `drop_mode` to 1 to always put the target on the ground.

### 8.3 Release steps

Do the steps in this sequence. Do a step only when its grab step is complete.

| Step | Call | Grab step |
| --- | --- | --- |
| 1 | Measurements (section 8.1) | 10 |
| 2 | `Target->K2_DetachFromActor(KeepWorld, KeepWorld, KeepWorld)` | 10 |
| 3 | If the placement rule applies (section 8.2): `K2_SetActorLocation((X, Y, ground Z + target half height + 10), false, hit, true)`. If this call returns false, use `K2_TeleportTo` with the same point. | 10 |
| 4 | When `release_upright` is 1, set the rotation to (0, yaw, 0) with `K2_SetActorRotation`. | None |
| 5 | `Target->SetDraggerCarrier(nullptr)`, only if `GetDraggerCarrier()` is still the carrier | 2 |
| 6 | Target CMC: `SetMovementMode(MOVE_Falling, 0)`, `Activate(true)`, `SetRequestedSpeed({0, 0, 0})` | 1 |
| 7 | Set `bIgnoreClientMovementErrorChecksAndCorrection` to the old value | 8 |
| 8 | Set the capsule response to `ECC_Pawn` to the old value | 3 |
| 9 | Set `bKnockdownImmune` to the old value | 9 |
| 10 | `RemoveActiveGameplayEffect(handle, -1)`. If it returns false, use `RemoveActiveGameplayEffectBySourceEffect(GE_GrantLungedTag_C, target ASC, -1)`. | 4 |
| 11 | If the target state is still 53, set `AS_PounceRecoverySmall` (24). After `reset_delay_ms` (default 1000), call `ResetAbilityState()` if the target is alive and still in state 24. | 7 |
| 12 | Carrier: if it is alive and still in the carrier state, set `AS_None` (84) | 6 |

With `set_dragged_actor` 1, also set `DraggedActor` of the carrier to null. With `damage_causer` 1, also set `LastDamagedTarget` of the carrier and `MyDamageCauser` of the target to null.

`ResetAbilityState()` sets `AS_None`.

If the mod has no timer for step 11, call `StartTimerToGoFromPinnedToNormal(1.0)`. This game function sets state 24 and resets the state after 1 second. The tested mod uses this function at server shutdown. The T. rex code also calls this function on one of its release paths.

```cpp
bool IsDead(SDK::ATICharacterBase* C)
{
    return C->bIsDead || C->GetAbilityState() == SDK::EAbilityState::AS_Corpse;
}

void ReleaseSteps(const GrabRecord& Rec, SDK::ATICharacterBase* Carrier, SDK::ATICharacterBase* Target,
                  SDK::UClass* LungedEffect, bool PlaceOnGround, double GroundZ)
{
    SDK::UTICharacterMovementComponentBase* Cmc = Target->CharacterMovementComponent;
    auto* Capsule = static_cast<SDK::UCapsuleComponent*>(Target->K2_GetRootComponent());
    SDK::UAbilitySystemComponent* Asc = SDK::UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(Target);
    const SDK::FVector Loc = Target->K2_GetActorLocation();
    const double HalfHeight = Capsule->GetScaledCapsuleHalfHeight();

    if (Rec.Done & StepAttach)
    {
        Target->K2_DetachFromActor(SDK::EDetachmentRule::KeepWorld, SDK::EDetachmentRule::KeepWorld, SDK::EDetachmentRule::KeepWorld);
        if (PlaceOnGround)
        {
            const SDK::FVector Dest{ Loc.X, Loc.Y, GroundZ + HalfHeight + 10.0 };
            SDK::FHitResult SweepHit{};
            if (!Target->K2_SetActorLocation(Dest, false, &SweepHit, true))
                Target->K2_TeleportTo(Dest, Target->K2_GetActorRotation());
        }
    }

    SDK::FRotator Upright{};
    Upright.Yaw = Target->K2_GetActorRotation().Yaw;
    Target->K2_SetActorRotation(Upright, false);

    if ((Rec.Done & StepCarrier) && Target->GetDraggerCarrier() == Carrier)
        Target->SetDraggerCarrier(nullptr);
    if (Rec.Done & StepMove)
    {
        Cmc->SetMovementMode(SDK::EMovementMode::MOVE_Falling, 0);
        Cmc->Activate(true);
        Cmc->SetRequestedSpeed(SDK::FVector{ 0.0, 0.0, 0.0 });
    }
    if (Rec.Done & StepCorrection)
        Cmc->bIgnoreClientMovementErrorChecksAndCorrection = Rec.OldIgnoreCorrection;
    if (Rec.Done & StepCapsule)
        Capsule->SetCollisionResponseToChannel(SDK::ECollisionChannel::ECC_Pawn, Rec.OldPawnResponse);
    if (Rec.Done & StepKnockdown)
        Cmc->bKnockdownImmune = Rec.OldKnockdownImmune;
    if ((Rec.Done & StepEffect) && !Asc->RemoveActiveGameplayEffect(Rec.Effect, -1))
        Asc->RemoveActiveGameplayEffectBySourceEffect(SDK::TSubclassOf<SDK::UGameplayEffect>(LungedEffect), Asc, -1);
    if ((Rec.Done & StepTargetState) && Target->GetAbilityState() == SDK::EAbilityState::AS_Crushed)
        Target->SetAbilityState(SDK::EAbilityState::AS_PounceRecoverySmall);
    if ((Rec.Done & StepCarrierState) && Carrier && !IsDead(Carrier)
        && Carrier->GetAbilityState() == SDK::EAbilityState::AS_CrushPoseStarted)
        Carrier->SetAbilityState(SDK::EAbilityState::AS_None);
}

void CheckedReset(SDK::ATICharacterBase* Target)
{
    if (Target->GetIsAlive() && Target->GetAbilityState() == SDK::EAbilityState::AS_PounceRecoverySmall)
        Target->ResetAbilityState();
}
```

The example shows the normal case. Before each call, make sure that each stored object is still valid (section 1.3). Call `CheckedReset` from the reset timer job.

### 8.4 Special cases

| Case | Do these steps |
| --- | --- |
| Target gone | Do not call the target. Do only step 12. |
| Takeover: `GetDraggerCarrier()` of the target is a different actor | A different holder has the target, for example a T. rex. Do only steps 9, 10 and 12. The tests did not include this case. |
| Target dead | Do steps 2, 5, 7, 8, 9, 10 and 12. Do not move the corpse. Do not change its state. |
| Carrier gone or dead | Do all target steps. Do step 12 only for a live carrier. |
| A grab step failed | Do only the steps of the completed grab steps. |

### 8.5 Check after the release

These checks are optional. The tested mod did them 1.5 seconds after each release, for tests. If `reset_delay_ms` + 500 ms is longer, it used that time.

- The target state is not 53 and not 24.
- `GetDraggerCarrier()` is null, and `GetAttachParentActor()` is null.
- The bit `bIgnoreClientMovementErrorChecksAndCorrection` and the capsule response have their old values.
- The target has no active `GE_GrantLungedTag_C` effect.
- The target CMC is active, and `IsFalling()`, `IsMovingOnGround()`, `IsSwimming()` or `IsFlying()` is true.

For a dead target, check only these items: the state is 30, and `GetAttachParentActor()` is null.

## 9. Logout, destroyed actors, settings change and shutdown

- Logout (`K2_OnLogout`) or safe logout (`PrepareSafeLogout`): release each grab of this controller or its pawn. A safe logout of the target player uses the placement rule of a release that the carrier caused (section 8.2). Thus the target falls.
- `K2_OnLogout` can come after the game destroys the pawn of the player. If this pawn is the target, the target is gone (section 8.4). Then the mod sets the recovery mark on the entry (section 10).
- Destroyed actor: release the grab with "target gone" or "carrier gone".
- Settings change: when a settings change sets `enabled` to 0, release all grabs. A grab that is active keeps its own copy of the settings (section 6).
- Settings error: the settings file can have an error while `enabled` is 1. Then the tested mod sets `enabled` to 0 in memory and releases all grabs.
- Shutdown: release all grabs on the game thread before the mod stops. Put each target on the ground. Use `StartTimerToGoFromPinnedToNormal(1.0)` for the state reset (section 8.3).

## 10. Player recovery entry (optional)

The recovery entry is for a player target. The game can destroy the target pawn during a grab. At the next login of this player, the mod moves the new pawn to a ground point. The tests did not include this case.

1. At each grab of a player target, write an entry to a file. The entry has these values:
   - The Steam ID and the class of the target. Read the Steam ID from `ATIPlayerController::SteamId` (17 digits).
   - The Steam ID and the class of the carrier.
   - The grab time, a ground point and the target half height.
2. For the ground point, trace down from the top of the target capsule (section 8.1, step 3). If the trace finds no ground, or the ground is water, use the lowest point of the target.
3. Update the point at the footsteps events of the carrier and at the target events. Update it only when the horizontal distance from the carrier to the point is more than 2000 cm. Use the ground below the carrier. Do not use a water point.
4. At a release, remove the entry. The exception is step 5.
5. If the target is not valid at the release, set the recovery mark on the entry. Do this only when grab step 10 is complete.
6. When the server starts, read the file. Remove the entries that are more than 24 hours old or more than 60 seconds in the future. If two entries have the same target Steam ID, remove the second entry. Set the recovery mark on all other entries.
7. While an entry has the recovery mark, keep the `UFunction` pointer of `APlayerController::ClientRestart` in your list (section 1.1). At each call, find the entry for the Steam ID of the controller.
8. If the class of `NewPawn` is the target class in the entry, move the pawn to the ground point. The new Z is the ground Z + the half height + 10 cm. Then call `StopMovementImmediately()`. If the class is different, do not move the pawn.
9. Remove the entry. When no entry has the recovery mark, remove the pointer from the list.

At shutdown, keep only the entries with the recovery mark.

## 11. Legs hang point (optional)

With the legs hang point, the target hangs below the body of the carrier and not from the mouth.

1. After the grab, wait for the larger of these two times:
   - `legs_delay_ms` (default 500).
   - Two network updates of the target: 2000 / `NetUpdateFrequency` ms, a maximum of 5000 ms.
2. Do the checks in section 7.2 again. Also make sure that the target is still attached to `DragLocation`. If a check fails, do not use the legs point.
3. Calculate the point in the local space of the carrier capsule:
   - X = `legs_x_cm` (default -45).
   - Y = `legs_y_cm` (default 0).
   - Z = `legs_feet_z_cm` (default -55) - (target half height + `legs_gap_cm`) / the Z scale of the carrier capsule.
   - The rotation is (`legs_pitch`, `legs_yaw`, `legs_roll`), default (0, 0, 0).
4. Select the parent. With `legs_parent` 2, the parent is the carrier capsule. Use the point and the rotation from step 3. With `legs_parent` 0 (default), the parent is the carrier mesh. Change the point and the rotation into the space of the mesh. Use the relative location, rotation and scale of the mesh.
5. Find the ground below the world position of the point. Start the trace at the carrier Z. The lowest point of the target must be 50 cm or more above the ground.
6. If step 5 fails or finds no ground, do steps 2 to 5 again later. Wait 1 second before the second try. Wait 2, 4 and 8 seconds before the next tries. If the fifth try also fails, keep the target at the mouth.
7. Attach the target to the parent with `K2_AttachToComponent(Parent, NAME_None, SnapToTarget, SnapToTarget, KeepWorld, false)`.
8. Set the relative point and rotation of the target root with `K2_SetRelativeLocationAndRotation(Point, Rotation, false, Hit, true)`.
9. Read back: the parent of the target root is the parent from step 4, and `GetAttachParentActor()` is the carrier. If not, attach the target to `DragLocation` again.
10. Change the relative Z 4 times. Use the legs Z + 0.1 cm for the first and third change. Use the legs Z - 0.1 cm for the second and fourth change. Make the changes 1, 3, 7 and 15 seconds after step 9 (`legs_nudge_ms` 1000, doubled each time).

The reason for step 10: on a client, the state-53 code attaches the target to `DragLocation`. The attachment from the server attaches it to the legs point. The sequence of these two updates on a client is not known. A small change of the relative point makes the server send the attachment again.

With the legs hang point, the rule "Distance" (section 7.2) uses the legs point. The rule "Legs parent" also applies.

## 12. Fish catch (optional)

### 12.1 Method

The fish in the water are `ATIFishSchool` actors (fish schools). The game gives a caught fish as a food item (the fish item). The mod does the same steps as the game, on the server only.

The tested mod does not call `ATIFishSchool::ServerAttemptBite`. A static analysis of the server code gives these results:

- For a carrier of the flying type (`ET_Volant`), `ServerAttemptBite` starts a pickup ability that waits for a client step. The analysis assumed that the Pteranodon has this type.
- `GA_Skim_C` blocks this ability. The same right-click starts `GA_Skim_C`.
- The game can destroy the fish school and give no fish item.

### 12.2 When to search for a fish school

Search for a fish school when `allow_fish_target` is 1 and one of these conditions is true:

- The target search (section 5) found no target.
- The number of grabs is at `max_active`.

A fish catch makes no grab record.

Do not catch a fish when the carrier is a hatchling (`IsHatchling()`). Also do not catch a fish when the carrier state is 2, 52 or 53.

### 12.3 Fish school search

1. Find the object type of the Interactable channel (`ECC_GameTraceChannel4`). Read `UCollisionProfile::GetDefaultObj()->DefaultChannelResponses`. The Interactable channel must be an object channel, not a trace channel.
2. Count the object channels from `ECC_EngineTraceChannel1` to the channel before the Interactable channel. The `EObjectTypeQuery` value is 6 + this count. The value 0 is `ObjectTypeQuery1`. On game version 0.21.784, the result is `ObjectTypeQuery9`.
3. Sweep with the same start, end, lead and ignore rules as the target search (section 5.1). Use the radius `fish_radius_cm` (default 1200) and the Interactable object type.
4. Accept only an `ATIFishSchool` actor. The object type of the hit component must be `ECC_GameTraceChannel4`. Add each other actor to the list of actors to ignore. Then sweep again.
5. The distance from the carrier to the fish school must be 1500 cm or less. The game uses the same limit.
6. Do the line-of-sight check to the location of the hit component (section 5.4).

The fish schools move away from a player. In the tests, the fish schools stayed 900 cm or more from the player. A small radius does not find them. Thus the default radius is large.

### 12.4 Fish item

1. Use `FishToSpawn` of the fish school. It must be a subclass of `ATIGoreBase`.
2. Some fish schools have no fish item, for example the catfish. With `fish_catfish_item` 1, use the fish item of the Coelacanth (`BP_Coalecanth_C`). Read `FishToSpawn` from its class default object.
3. If no fish item exists, do not catch the fish.

### 12.5 Catch steps

| Step | Call |
| --- | --- |
| 1 | Carrier: set `SchoolFishToSpawn` to the item class. Set `bScaleSchoolFishPickupToCharacter` to `bScaleCaughtFishToCharacter` of the fish school. |
| 2 | `Carrier->SpawnPieceOfMeat(ItemClass, false, true, false, 0, 0, 0, {1, 1, 1}, true, nullptr, nullptr)` |
| 3 | Find the fish item with `Carrier->GetDraggedPickablePiece()`. It must be a valid object (section 1.3) of the item class. |
| 4 | Weight: if the item `GetWeight()` is more than the carrier `GetWeight()`, the fish item is heavy. With `fish_heavy_hold` 1 (default), continue. With 0, call `Item->DropOnTheFloor(Carrier, false)`. Then call `School->K2_DestroyActor()`. Do not do steps 5 to 11. |
| 5 | `Item->HandlePickUpEvent(false, Item->GetActorScale3D(), 0, true)` and `Item->SetIsPickedUp(true)` |
| 6 | `Item->K2_AttachToComponent(Carrier->DragLocation, "PickUpSocket", SnapToTarget, SnapToTarget, KeepWorld, true)`. The attach parent must be the carrier. |
| 7 | `Item->SetOwner(Carrier)` and `Carrier->SetNearbyPickablePiece(nullptr)` |
| 8 | `Carrier->SetAbilityState(AS_HoldingObject)` (2). The state must be 2. |
| 9 | `Carrier->SetMouthGoreValue(1.0)`, `Item->ForceNetUpdate()` and `Carrier->ForceNetUpdate()` |
| 10 | `School->K2_DestroyActor()` |
| 11 | Cancel each active `UTIGameplayAbilitySkim` instance of the carrier with `K2_CancelAbility()` |

If step 3 fails, set `DraggedPickablePiece` and `NearbyPickablePiece` of the carrier to null. If step 6 or step 8 fails, also detach the fish item. Then destroy it.

An adult Pteranodon has a weight of about 90 to 120. The Coelacanth fish item has a weight of 300. Thus the heavy rule applies to this fish item.

### 12.6 Fish item release

With `fish_rmb_drop` 1, the release input of the carrier drops the fish item (section 3):

1. `Item->K2_DetachFromActor(KeepWorld, KeepWorld, KeepWorld)`.
2. `Item->ResetGore()` and `Item->FlushNetDormancy()`.
3. `Carrier->SetDraggedPickablePiece(nullptr)`.
4. If the carrier is alive, its state is 2, `IsStunned()` is false and `IsPounced()` is false, set `AS_None`.

Do the same steps when the carrier dies or leaves the game. Do step 4 only for a live carrier. When the game destroys the carrier, do only steps 1 and 2.

```cpp
bool GiveFish(SDK::ATICharacterBase* Carrier, SDK::ATIFishSchool* School, SDK::UClass* ItemClass,
              SDK::FName PickUpSocket, bool HeavyHold)
{
    Carrier->SchoolFishToSpawn = SDK::TSubclassOf<SDK::ATIGoreBase>(ItemClass);
    Carrier->bScaleSchoolFishPickupToCharacter = School->bScaleCaughtFishToCharacter;
    Carrier->SpawnPieceOfMeat(SDK::TSubclassOf<SDK::ATIGoreBase>(ItemClass), false, true, false, 0.0f, 0.0f, 0.0f,
                              SDK::FVector{ 1.0, 1.0, 1.0 }, true, nullptr, nullptr);
    SDK::ATIGoreBase* Item = Carrier->GetDraggedPickablePiece();
    if (!Item || !Item->IsA(ItemClass))
        return false;
    if (Item->GetWeight() > Carrier->GetWeight() && !HeavyHold)
    {
        Item->DropOnTheFloor(Carrier, false);
        School->K2_DestroyActor();
        return true;
    }
    Item->HandlePickUpEvent(false, Item->GetActorScale3D(), 0.0f, true);
    Item->SetIsPickedUp(true);
    if (!Item->K2_AttachToComponent(Carrier->DragLocation, PickUpSocket, SDK::EAttachmentRule::SnapToTarget,
                                    SDK::EAttachmentRule::SnapToTarget, SDK::EAttachmentRule::KeepWorld, true)
        || Item->GetAttachParentActor() != Carrier)
        return false;
    Item->SetOwner(Carrier);
    Carrier->SetNearbyPickablePiece(nullptr);
    Carrier->SetAbilityState(SDK::EAbilityState::AS_HoldingObject);
    if (Carrier->GetAbilityState() != SDK::EAbilityState::AS_HoldingObject)
        return false;
    Carrier->SetMouthGoreValue(1.0f);
    Item->ForceNetUpdate();
    Carrier->ForceNetUpdate();
    School->K2_DestroyActor();
    return true;
}
```

Make `PickUpSocket` one time with `UKismetStringLibrary::Conv_StringToName(FString(L"PickUpSocket"))`. The example does not show the undo steps and the skim cancel.

## 13. Settings

Read the settings from a text file. Use a file change notification, for example `ReadDirectoryChangesW`. Do not poll the file. After a change, read the file two times. Apply the change only when the two results are equal.

The table gives the values in the default settings file of the tested mod. It does not show the log settings, the admin list and `watchdog_ms`. The mod does not use `watchdog_ms`.

| Setting | Default | Range | Purpose |
| --- | --- | --- | --- |
| `enabled` | 0 | 0 to 1 | The mod is on (1) or off (0) |
| `trigger_mode` | 1 | 0 to 1 | 1 = all players can grab. 0 = a trigger input does not start a grab. |
| `trigger_match` | 0 | 0 to 2 | 0 = latch or skim ability, 1 = input ID 2 or 3, 2 = all trigger inputs |
| `release_input` | 1 | 0 to 1 | 1 = a trigger input releases the target |
| `brake_window_ms` | 400 | 0 to 5000 | Time after a brake change, for a trigger input |
| `press_dedupe_ms` | 300 | 0 to 5000 | Discard a second trigger input in this time |
| `cooldown_ms` | 1000 | 0 to 60000 | Minimum time between two accepted trigger inputs of one carrier |
| `carrier_state` | 5 | 2, 4 or 5 | Ability state of the carrier |
| `set_dragged_actor` | 0 | 0 to 1 | 1 = also set `DraggedActor` of the carrier |
| `damage_causer` | 0 | 0 to 1 | 1 = also set `MyDamageCauser` of the target and `LastDamagedTarget` of the carrier |
| `pickup_event` | 0 | 0 to 1 | 1 = call `PickupEventFromCrush()` on the target after grab step 10 |
| `apply_ge` | 1 | 0 to 1 | Grab step 4 |
| `cancel_abilities` | 1 | 0 to 1 | Grab step 5 |
| `requested_speed_zero` | 1 | 0 to 1 | `SetRequestedSpeed({0, 0, 0})` in grab step 1 |
| `knockdown_immune` | 1 | 0 to 1 | Grab step 9 |
| `attach_delay_ms` | 0 | 0 to 500 | Delay before grab step 10 |
| `release_mode` | 2 | 0 to 2 | State reset: 2 = checked reset (section 8.3), 1 = set `AS_None` immediately, 0 = `StartTimerToGoFromPinnedToNormal(1.0)` |
| `reset_delay_ms` | 1000 | 200 to 10000 | Delay of the checked reset |
| `release_upright` | 1 | 0 to 1 | Release step 4 |
| `drop_mode` | 0 | 0 to 1 | 1 = always put the target on the ground |
| `reach_cm` | 300 | 50 to 600 | Maximum gap below the carrier |
| `radius_cm` | 120 | 20 to 400 | Radius of the target sweep |
| `lead_s` | 0.10 | 0 to 0.5 | Velocity lead time |
| `lead_max_cm` | 150 | 0 to 500 | Maximum lead length |
| `max_sweeps` | 4 | 1 to 4 | Maximum sweeps for one trigger input |
| `los_check` | 1 | 0 to 1 | Line-of-sight check |
| `target_require_as84` | 0 | 0 to 1 | 1 = grab only targets in `AS_None` |
| `allow_flying_target` | 0 | 0 to 1 | 1 = the mod accepts a target in flight |
| `allow_swimming_target` | 0 | 0 to 1 | 1 = the mod accepts a target in the water |
| `allow_ai_target` | 1 | 0 to 1 | 1 = the mod accepts an AI target |
| `weight_ratio` | 0 | 0 to 10 | 0 = no weight rule |
| `max_hold_ms` | 60000 | 1000 to 600000 | Maximum hold time |
| `max_dist_cm` | 100 | 20 to 1000 | Maximum distance from the attach point |
| `max_active` | 2 | 1 to 4 | Maximum number of grabs at the same time |
| `drop_on_land` | 1 | 0 to 1 | Rules "carrier landed" and "carrier mode" |
| `drop_on_hit` | 1 | 0 to 1 | Rules "carrier crashed" and "carrier hit" |
| `drop_on_touch` | 1 | 0 to 1 | Rule "carrier touch" |
| `drop_on_water` | 1 | 0 to 1 | Rule "carrier water" |
| `drop_on_victim_hit` | 1 | 0 to 1 | Rule "target hit" |
| `land_grace_ms` | 500 | 0 to 10000 | Land grace time (section 7.3) |
| `hit_grace_ms` | 500 | 0 to 10000 | Hit grace time (section 7.3) |
| `victim_hit_grace_ms` | 1500 | 0 to 10000 | Target hit grace time (section 7.3) |
| `hang_point` | 0 | 0 to 1 | 0 = mouth, 1 = legs |
| `legs_parent` | 0 | 0 or 2 | 0 = mesh, 2 = capsule |
| `legs_delay_ms` | 500 | 100 to 5000 | Section 11, step 1 |
| `legs_nudge_ms` | 1000 | 250 to 5000 | Section 11, step 10 |
| `legs_x_cm`, `legs_y_cm`, `legs_feet_z_cm` | -45, 0, -55 | -300 to 300 | Legs point |
| `legs_gap_cm` | 0 | -100 to 300 | Additional gap below the legs point |
| `legs_pitch`, `legs_yaw`, `legs_roll` | 0 | -180 to 180 | Rotation of the target at the legs point |
| `allow_fish_target` | 1 | 0 to 1 | Fish catch |
| `fish_heavy_hold` | 1 | 0 to 1 | 1 = keep a heavy fish item |
| `fish_catfish_item` | 1 | 0 to 1 | 1 = the Coelacanth fish item for a fish school with no fish item |
| `fish_rmb_drop` | 1 | 0 to 1 | 1 = the release input drops the fish item |
| `fish_radius_cm` | 1200 | 100 to 1500 | Radius of the fish school sweep |

The tested mod also has admin commands: status, release one player, and release all grabs.

## 14. Important notes

- With `drop_mode` 0, a target can fall from a large height (section 8.2).
- Make the SDK again after each game update. Then check the hook functions, the two movement bits, the `bHasBegunPlay` byte and the class names again.

## 15. Test status

Test conditions: game version 0.21.784, a dedicated test server, and a player Pteranodon as the carrier. Most tests used `reach_cm` 600 and `radius_cm` 300. The tests of targets in flight and in the water used `allow_flying_target` 1 and `allow_swimming_target` 1.

These parts of the mod worked in the tests:

| Part | Result |
| --- | --- |
| Trigger | The server received `ServerSetIsAirBraking` for each brake change. It received two trigger inputs for each right-click (section 2.5). |
| AI targets | The tests used AI Diabloceratops, Dryosaurus and Hypsilophodon targets. The release check (section 8.5) passed. Two early failures came from errors in an older version of the check. |
| Carrier screen | The target hung from the mouth. The carrier flew normally. |
| Release causes | These releases worked: release input, "carrier landed" (footsteps event), "carrier hit" (a tree, `OnCapsuleHit`), "carrier crashed" and carrier death. "Carrier landed" during the grace time also worked (grace timer). |
| Placement | In one release, the target was 86 cm below the terrain. The mod moved it back to the ground. |
| Legs hang point | The target hung below the body, and the carrier flew normally. After one grab near the ground, the target stayed at the mouth. |
| Player targets | The tests used a player Deinosuchus in the water (5 grabs) and a player Pteranodon in flight (1 grab). The server grab steps and release steps were correct. One release check failed because an older check did not count flight as movement. That player flew normally after the release. |
| Water | The mod grabbed targets in the water and in a river with the line-of-sight water rule. |
| Fish | The mod caught one Coelacanth with a heavy fish item. A release input dropped the fish item. |

