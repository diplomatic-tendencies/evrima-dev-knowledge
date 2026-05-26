# EVRIMA C++ side mods (UE4SS toolchain)

C++ side mods are the right tool for capabilities Lua cannot reach. The Lua-mod surface gets you a long way (every mod in this knowledge bundle is pure Lua), but specific things genuinely require C++: hooking C++ virtuals that aren't UFunctions, defining new UCLASSes, declaring custom multicast RPCs, reading FString fields on USTRUCTs that crash UE4SS Lua marshaling.

This document covers the verified end-to-end toolchain for building UE4SS C++ side mods against The Isle EVRIMA.

## What C++ unlocks that Lua doesn't

The capability boundary, in order of practical value:

1. **Hook C++ virtuals (not just UFunctions).** The damage-effect pipeline runs through `PostGameplayEffectExecute` on the AttributeSet, which is a C++ virtual not a UFUNCTION. Lua reflection cannot reach it. C++ can install a vtable hook for clean GAS-level damage interception (no polling lag, catches DoT and environmental kills).

2. **Read FString fields on USTRUCTs.** Lua native rule 2 prevents naming FString fields on certain structs. `TIPlayerData` has FString fields (Class, SteamId, MateName) that crash if accessed via Lua field syntax. C++ can read by offset using values from the UHT dump.

3. **Declare new UCLASSes.** Lua spawns existing BP classes. C++ can define new C++ classes (e.g. subclass `ATICharacterBase` with custom AI logic) and Lua can then spawn instances of those new classes.

4. **Multicast RPCs server-to-client.** Lua can't cleanly create new multicast RPCs. C++ can declare them with proper replication. Custom client-side UI popups, admin banners with formatting, server messages outside the chat box.

5. **Custom replicated UFunctions.** Declare your own C++ UFunctions with validation and replication. Call them from Lua. Cleaner bot architecture (a UFunction call over network instead of file-based IPC).

6. **Engine tick callbacks.** Lua's `LoopInGameThreadWithDelay` is minimum 1-second practical granularity. C++ can attach to actor ticks at sub-frame resolution. Frame-perfect timing, real-time state monitoring.

7. **`.pak` content plus C++ spawner wiring.** Server-side Lua cannot make new visual content render client-side (mesh field doesn't replicate; Niagara needs a renderer the dedicated server lacks). The solution is `.pak` asset packs containing custom BPs (built in Unreal Editor) with meshes and particles baked into the class default. A C++ side mod can spawn instances of those custom BP classes.

8. **Bypass TArray-of-FStruct marshaling pain.** Some UE APIs return wrapped TArrays where each element needs defensive unwrapping. C++ accesses TArray elements directly with full type info.

## What C++ does NOT solve

Most engine bugs are not Lua bugs. They are engine bugs. C++ calling the same UFunctions hits the same engine bugs:

- The SetSlot batching limit (only the last call in a tick commits) is engine-side; C++ hits it too.
- The quest-mutation validation gate is engine-side.
- The GAS attribute auto-fill on growth change is engine behavior.
- `K2_DestroyActor` on a freed actor still crashes in C++ if you don't use the right UE reference primitives (TWeakObjectPtr, IsValid).

C++ avoids the UE4SS Lua marshaling layer specifically. Engine-level bugs persist.

## Toolchain prerequisites

Verified working as of mid-2026:

| Component | Version | Notes |
|---|---|---|
| Visual Studio 2022 Community | 17.13+ | Free. Free for individual use. |
| MSVC C++ Tools | 14.43+ | C++23 support required. |
| Windows 11 SDK | 10.0.26100 | The one that ships with VS 2022 17.13. |
| Rust toolchain | 1.73+ | Used internally by UE4SS for tooling. Install via rustup. |
| CMake | 3.22+ | Build system. 4.x works. |
| Git for Windows | recent | Plus Git Credential Manager. |
| Epic Games account linked to GitHub | n/a | Required for the UEPseudo submodule. Free signup. |

The total toolchain footprint is roughly 15 to 20 GB. NOT a full Unreal Engine install; UE4SS builds against its own headers plus a small UE pseudo-header dump.

## Build steps

```bash
# Identify the installed UE4SS version. UE4SS.log first line on boot reads:
#   "UE4SS - vX.Y.Z Beta #N - Git SHA #abcdef12"
# Match that commit when cloning.

cd <workdir>
git clone https://github.com/UE4SS-RE/RE-UE4SS.git ue4ss-src
cd ue4ss-src
git checkout <commit-from-log>

# After Epic linkage is set up:
git config --global url."https://github.com/".insteadOf "git@github.com:"
gh auth setup-git  # if using gh CLI for auth
git submodule update --init --recursive

# Configure
cmake -B build_cmake_Game__Shipping__Win64 -G "Visual Studio 17 2022" -A x64

# Or with Ninja for faster builds:
# cmake -B build_cmake_Game__Shipping__Win64 -G Ninja -DCMAKE_BUILD_TYPE=Game__Shipping__Win64

# Build (matches installed mode)
cmake --build build_cmake_Game__Shipping__Win64 --config Game__Shipping__Win64
```

The build produces:
- `<build>/Game__Shipping__Win64/bin/UE4SS.dll` (the runtime; you typically use the existing installed one)
- `<build>/Game__Shipping__Win64/lib/UE4SS.lib` (the import library that cppmods link against)
- `<build>/Game__Shipping__Win64/bin/dwmapi.dll` (the proxy DLL)
- Plus two example cppmod DLLs (EventViewerMod, KismetDebuggerMod) as reference

## UEPseudo: the Epic-gated submodule

The `deps/first/Unreal` submodule points to a private repo (`Re-UE4SS/UEPseudo`) that requires an Epic-Games-linked GitHub account to clone. This is the same access model Epic uses for the UnrealEngine repo itself.

## Scaffolding a new C++ side mod

The minimum structure:

```
cppmods/MyMod/
├── CMakeLists.txt          # 10 lines
└── src/
    └── dllmain.cpp         # CppUserModBase subclass + start_mod/uninstall_mod
```

`CMakeLists.txt`:

```cmake
set(TARGET MyMod)
project(${TARGET})
set(${TARGET}_Sources "${CMAKE_CURRENT_SOURCE_DIR}/src/dllmain.cpp")
add_library(${TARGET} SHARED ${${TARGET}_Sources})
target_link_libraries(${TARGET} PUBLIC UE4SS)
```

`src/dllmain.cpp` (minimal hello-world):

```cpp
#include <DynamicOutput/Output.hpp>
#include <Mod/CppUserModBase.hpp>

using namespace RC;

class MyMod : public CppUserModBase
{
  public:
    MyMod() : CppUserModBase()
    {
        ModName = STR("MyMod");
        ModVersion = STR("0.1.0");
        ModDescription = STR("Hello-world C++ side mod");
        ModAuthors = STR("you");
        ModIntendedSDKVersion = STR("3.0.1");
        Output::send<LogLevel::Verbose>(STR("[MyMod] constructor\n"));
    }

    auto on_unreal_init() -> void override
    {
        Output::send<LogLevel::Verbose>(STR("[MyMod] on_unreal_init\n"));
    }

    auto on_program_start() -> void override
    {
        Output::send<LogLevel::Verbose>(STR("[MyMod] on_program_start\n"));
    }

    auto on_update() -> void override {}
};

#define MYMOD_API __declspec(dllexport)

extern "C"
{
    MYMOD_API CppUserModBase* start_mod() { return new MyMod(); }
    MYMOD_API void uninstall_mod(CppUserModBase* mod) { delete mod; }
}
```

Then add one line to `cppmods/CMakeLists.txt`: `add_subdirectory("MyMod")`.

Reconfigure CMake (`cmake -B ...` again, takes about 40 seconds for incremental) and build just your target:

```bash
cmake --build build_cmake_Game__Shipping__Win64 \
      --config Game__Shipping__Win64 --target MyMod
```

Subsequent builds of just your mod take roughly 10 seconds.

## Deployment

UE4SS loads cppmods from `<game>/TheIsle/Binaries/Win64/Mods/<ModName>/dlls/main.dll`. The file MUST be named `main.dll` (NOT `MyMod.dll`). It MUST export `start_mod` and `uninstall_mod` as `extern "C"`. Enable in `Mods/mods.txt` with `MyMod : 1`.

UE4SS logs `Starting C++ mod 'MyMod'` on boot when it picks up the entry.

## CppUserModBase lifecycle

Verified order observed during live boot:

1. **constructor** runs early, BEFORE on_program_start. Set metadata here (ModName, ModVersion, etc.). `Output::send` works immediately.
2. **on_unreal_init()**: the Unreal:: namespace is now safe to use. This is where you install hooks on UClass, UFunction, or virtuals.
3. **on_program_start()**: fires LAST in the boot sequence, not first. Don't trust the name to mean "earliest hook."
4. **on_update()**: every tick. On the EVRIMA dedicated server this fires at about 160 Hz, NOT 30 Hz as you might guess from the server tick rate. Throttle aggressively or use hook-based callbacks instead.
5. **destructor**: on shutdown or hot-reload.

## Build gotchas (verified during real cppmod work)

**`#include <Windows.h>` requires `WIN32_LEAN_AND_MEAN` plus `NOMINMAX` defined BEFORE the include.** Otherwise MSVC errors out on UE4SS's internal Hooks header with `error C2131: expression did not evaluate to a constant ... see usage of 'Lowest'`. The Windows macros shadow internal UE4SS symbols. The pattern:

```cpp
#ifndef WIN32_LEAN_AND_MEAN
#define WIN32_LEAN_AND_MEAN
#endif
#ifndef NOMINMAX
#define NOMINMAX
#endif
#include <Windows.h>
```

**`Output::send<LogLevel::X>(STR("..."), args...)` uses the fmt library with a WIDE format context.** Passing `const char*` for a `{}` placeholder produces a multi-screen template error in `fmt/base.h`. Use `L"..."` literals, not `"..."`. The `STR()` macro produces `L"..."` on Windows.

**Deprecated forwarding includes** that emit build-time warnings but still work: `<Unreal/UClass.hpp>`, `<Unreal/UFunction.hpp>`. Use `<Unreal/CoreUObject/UObject/Class.hpp>` instead (covers both).

**`StringType` is in the `RC::` namespace, not `RC::Unreal::`.** A `using namespace RC;` at the top of dllmain.cpp brings it into scope.

**DLL deployment requires killing BOTH `TheIsleServer` and `TheIsleServer-Win64-Shipping` processes.** `Stop-Process -Name "TheIsleServer"` does NOT match the actual binary `TheIsleServer-Win64-Shipping`. Use a regex match or stop both names. Wait 8 to 10 seconds after stop before copying the DLL (Windows file lock release delay).

## C++ UFunction hook API

```cpp
#include <Unreal/UObjectGlobals.hpp>
#include <Unreal/UFunctionStructs.hpp>

using UnrealScriptFunctionCallable = std::function<
    void(UnrealScriptFunctionCallableContext& Context, void* CustomData)>;

// Context provides:
//   Context.Context  - UObject* self (the instance the function was called on)
//   Context.TheStack - FFrame& for reading the script execution stack (args)
//   Context.RESULT_DECL - void* for the return value slot

auto [pre_id, post_id] = UObjectGlobals::RegisterHook(
    StringType(STR("/Script/TheIsle.TICharacterBase:ApplyDamage")),
    [](UnrealScriptFunctionCallableContext& Ctx, void* CustomData) {
        // pre-hook: args not yet unpacked into Locals
        UObject* attacker = Ctx.Context;
    },
    [](UnrealScriptFunctionCallableContext& Ctx, void* CustomData) {
        // post-hook: args available in Ctx.TheStack.Locals()
        UFunction* fn = Ctx.TheStack.Node();
        uint8_t* locals = Ctx.TheStack.Locals();
        for (FProperty* param : TFieldRange<FProperty>(fn)) {
            if (param->HasAnyPropertyFlags(CPF_OutParm | CPF_ReturnParm)) continue;
            UObject* target = *param->ContainerPtrToValuePtr<UObject*>(locals);
            // First non-out param is Target; use it.
            break;
        }
    },
    nullptr
);
```

The string-based RegisterHook looks up the UFunction by full name. Returns `std::pair<int, int>` of (pre_id, post_id); both non-zero means success.

## C++ virtual hook (the hard one)

For C++ virtuals that are not UFunctions (like `PostGameplayEffectExecute` on UAttributeSet), you need either:

1. Vtable slot identification by hand. Look at a live instance's vtable, identify the target function's slot by cross-referencing pattern bytes (returns-true, returns-false, empty-ret stubs versus real implementations), then patch the slot via VirtualProtect plus pointer swap.

2. Signature scanning. Use a pattern-matching library (UE4SS includes patternsleuth as a submodule) to find the function's body in the binary by byte pattern, then patch.

Approach 1 is cheaper if the target class has a stable layout you can identify from a vtable dump. Approach 2 is more resilient to game updates but harder to set up.

Both approaches use PolyHook_2 (a UE4SS dependency) for the actual hook installation once you have the address.

## ABI compatibility

CppUserModBase.hpp explicitly notes: "C++ mods will break if UE4SS and the mod don't use the same C Runtime library version." This means:

- Same MSVC version (14.43+ matches recent UE4SS 3.0.x builds)
- Same configuration (Game__Shipping__Win64 matches installed UE4SS)
- Same UE4SS commit if possible (or at least same minor version)

Build from the exact commit reported by the installed UE4SS.dll boot log, not just the latest tag. The commit SHA is in the first log line.

## When to use C++ vs Lua

In practice, the right mental model is:

- **Lua** is the orchestration layer. Events, configs, persistence, chat, periodic ticks, file IO, NDJSON streams. Adequate for 80% of mod work.
- **C++** is the precision tool. Native virtuals, direct memory, custom classes, multicast RPCs, .pak wiring. Used surgically for specific capabilities, called from or alongside Lua.

Most production mods stay pure Lua. New advanced features come in as C++ side mods when they actually need the C++ capabilities. No premature C++; the iteration loss (no hot-reload, longer build cycles) is real.

## Closing notes

The toolchain has roughly 9 distinct gotchas to learn (Windows.h includes, fmt format width, deprecated header paths, StringType namespace, DLL kill sequence, on_update tick rate, ABI matching, the winget passive-install trap). Once they're in muscle memory, building a new C++ side mod is roughly:

1. Make the directory and files .
2. Add one line to `cppmods/CMakeLists.txt`.
3. Reconfigure CMake .
4. Build the target .
5. Deploy DLL to `Mods/<name>/dlls/main.dll`.
6. Restart server.

A hello-world cppmod loaded live in a verified end-to-end test on a fresh toolchain including all the initial dead-ends. 

The first session is the expensive one. After that, C++ side mods are routine.
