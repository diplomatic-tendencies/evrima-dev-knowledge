# How to Disable Tree Knockdown (The Isle: Evrima)

## Purpose

This document gives the procedure to disable the tree knockdown feature on a
dedicated server. The change is server-side. It does not change the game client.

## Application

- Game version: The Isle: Evrima 0.21.772.
- Tools: a server side mod that can read and write game objects. For example,
  UE4SS with Lua, or a C++ SDK mod.

## How the feature works

The server keeps the tree knockdown rules in one array. The array is a member of
the world settings object:

- Class: `ATIWorldSettings`
- Member: `TreeKnockdownSettings`
- Type: an array of `FTITreeKnockdownSettings`

Each item in the array is one rule for one type of tree. The Gateway map has 9
items. The server reads this array to find which trees can fall. If the array is
empty, no tree falls.

## Procedure

To disable the feature, empty the array. Do this one time, after the server loads
the world.

1. Wait until the world is available.
2. Get the `ATIWorldSettings` object.
3. Get the `TreeKnockdownSettings` array.
4. Set the number of items in the array to 0.

After this step, no tree falls.

## Important notes

- The map fills the array again each time the server loads the map. Do the change
  again after each server start. Latch the change, so it runs one time for each
  start.
- Do all game-object work on the game thread.
- Keep the member offset correct for each game patch. Use the typed member from a
  current SDK dump. Do not use a fixed raw address.
- The change has no measurable effect on the server frame rate. In test, the
  game-thread cost stayed near zero.

## Tool notes

- The class and the member are the same for all tools.
- For a C++ SDK mod, use the typed member `ATIWorldSettings::TreeKnockdownSettings`.
  Get the world settings object with `UWorld::GetWorld()->K2_GetWorldSettings()`.
- For a UE4SS Lua mod, get the world settings object, then empty the array through
  the Lua object interface.
