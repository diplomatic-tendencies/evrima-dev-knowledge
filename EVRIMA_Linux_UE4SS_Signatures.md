# UE4SS on the Linux dedicated server: AOB signatures and patch recovery

How UE4SS finds the engine on a stripped Linux EVRIMA server binary, what game build **0.22.157** (Steam build `26883482`) broke, and a repeatable way to recover the broken signature after future patches.

Everything here was verified live on a Linux dedicated server (Steam app `412680`, `evrima` branch, Pterodactyl / Docker, Ubuntu 24.04 host) running the Linux port of RE-UE4SS (`ocristopfer/RE-UE4SS`, release `linux-v3.3.3`, which reports itself in the log as `UE4SS - v4.0.0 Experimental - Git SHA #a939725`). The Windows server uses a different binary and different signatures; nothing in this document says anything about Windows.

## TL;DR for 0.22.157

- The patch changes `StaticConstructObject_Internal` enough that the reference signature no longer matches. UE4SS retries the Lua override scan about 1500 times (~2 minutes), then logs `Fatal Error: AOB scans could not be completed ... Was unable to find AOB for 'StaticConstructObject' via Lua script` and **no Lua mod loads**.
- **The game server keeps running normally**, so the break is silent: players can join, nothing crashes, and every mod is just gone. With auto-update on restart this can go unnoticed for hours. Check `ue4ss/UE4SS.log` after every update.
- The other three signatures (`FName_ToString`, `FName_Constructor`, `GNatives`) still match on 0.22.157.
- Working `UE4SS_Signatures/StaticConstructObject.lua` for 0.22.157 / build 26883482:

```lua
function Register()
    return "55 41 57 41 56 41 55 41 54 53 48 81 EC B8 02 00 00 49 89 FE 4C 8B 27 48 8B 47 08 48 89 44 24 40"
end

function OnMatchFound(MatchAddress)
    return MatchAddress
end
```

With it the log shows `StaticConstructObject_Internal address: 0x4a274b0 <- Lua Script`, the detour installs, and all Lua mods start. Re-verified on 0.22.157 after the fix, on an empty server: `RegisterHook` on `TIPlayerController:GetChatMessage` / `SetAdminCred`, `StaticFindObject` + `LoadAsset` resolution of dinosaur pawn classes and `BP_AI_*_Controller` classes, reflection reads on `TIAIWorldSpawner`, `KismetSystemLibrary:LineTraceSingle` from a game-thread loop, `DumpUSMAP()`, and RCON. Player-facing calls (`ClientShowNotification`, prime, AI `SpawnActor`) were not re-tested yet because nobody was online.

## Background: why the Linux build needs custom signatures

The Linux server binary `TheIsleServer-Linux-Shipping` is stripped (no `.sym`, no exported engine symbols), so UE4SS cannot resolve the engine by name. It needs four AOB signatures in `ue4ss/UE4SS_Signatures/`:

| File | What UE4SS needs it for |
|---|---|
| `FName_ToString.lua` | every name lookup |
| `FName_Constructor.lua` | building FNames from Lua strings |
| `StaticConstructObject.lua` | object construction, and the hook UE4SS uses to keep its object-search pools current. Without it UE4SS treats the scan as fatal. |
| `GNatives.lua` | Blueprint/UFunction calls from Lua (`OnMatchFound` returns `DerefToInt32(MatchAddress + 0x4)`) |

`GUObjectHashTables` and `ConsoleManagerSingleton` are reported as not found on every boot. That is harmless: UE4SS continues ("PS scan successful") and no tested feature depends on them.

The Linux release ships `LinuxReferencePacks.tar.gz` with per-engine-version reference patterns (`pack-5.6.json` for EVRIMA's UE 5.6). Those are patterns from *another* UE 5.6 game, so they are a starting point, not a guaranteed match. The approach that worked for the original setup: for every reference pattern, take its `min` prefix and extend it byte by byte until it is unique inside the executable segments of the EVRIMA binary.

```python
# sigmatch.py <TheIsleServer-Linux-Shipping>  (needs pack-5.6.json from LinuxReferencePacks)
import json, re, struct, sys
data = open(sys.argv[1], "rb").read()
phoff = struct.unpack_from("<Q", data, 32)[0]; phsz, phn = struct.unpack_from("<HH", data, 54)
rng = []
for i in range(phn):
    t, f, off, va, pa, fs, ms, al = struct.unpack_from("<IIQQQQQQ", data, phoff + i * phsz)
    if t == 1 and f & 1: rng.append((off, off + fs))
pack = json.load(open("LinuxReferencePacks/pack-5.6.json"))
def matches_at(pos, toks):
    return all(t in ("??", "?") or data[pos + j] == int(t, 16) for j, t in enumerate(toks))
for name, entries in pack["signatures"].items():
    for idx, e in enumerate(entries):
        toks = e["code"].split(); n = e["min"]
        rx = re.compile(b"".join(b"." if t in ("??", "?") else re.escape(bytes([int(t, 16)])) for t in toks[:n]), re.S)
        cands = [m.start() for a, b in rng for m in rx.finditer(data, a, b)]
        while len(cands) > 1 and n < len(toks):
            n += 1; cands = [c for c in cands if matches_at(c, toks[:n])]
        status = "unique" if len(cands) == 1 else f"{len(cands)} matches"
        print(f"{name}[{idx}] {status} len={n}" + (f" at 0x{cands[0]:x}" if len(cands) == 1 else ""))
        if len(cands) == 1: print("   ", " ".join(toks[:n]))
```

On 0.22.157 this prints unique matches for `FName_ToString`, `FName_Constructor` and `GNatives`, and `0 matches` for both `StaticConstructObject` entries (`GUObjectArray` also reports `0 matches`; UE4SS does not need it from the pack here).

## What changed in 0.22.157

The previous build (Steam build `24664709`) matched the second reference entry directly:

```
55 41 57 41 56 41 55 41 54 53 48 81 EC 58 02 00 00 48 89 FB 4C 8B 37 48      (build 24664709)
```

In 0.22.157 the function body is essentially the *first* reference entry again (same 0x2B8 stack frame, same `mov esi, 0x37` / `mov r8d, 1` / `mov r9d, 1` call sequence), but the compiler reallocated registers: the parameter struct pointer moved from `rbx` to `r14`, and the class pointer from `r14` to `r12`:

```
reference : 55 41 57 41 56 41 55 41 54 53 48 81 EC B8 02 00 00 48 89 FB 4C 8B 37 48 8B 47 08 ...
0.22.157  : 55 41 57 41 56 41 55 41 54 53 48 81 EC B8 02 00 00 49 89 FE 4C 8B 27 48 8B 47 08 ...
                                                               ^^^^^^^^ ^^^^^^^^
```

Byte patterns that pin register choices are exactly what breaks on a recompile, so expect this to happen again.

## Recovering StaticConstructObject after a patch, without guessing bytes

The function has a CPU-profiler scope named `UObject_StaticConstructObjectInternal`, and the scope name survives in the shipping binary as a UTF-16 string. That gives a code-independent anchor:

1. Find the UTF-16LE string `UObject_StaticConstructObjectInternal`.
2. The binary is non-PIE (image base `0x200000`), so the static initialiser that registers the scope loads the string with an absolute immediate: `mov edi, <scope object>` (`BF imm32`) directly followed by `mov esi, <string VA>` (`BE imm32`). The `edi` immediate is the address of the scope's static object.
3. Search the executable segments for RIP-relative references (`ModRM & 0xC7 == 0x05`) to that object. A few small helpers touch it too; the real `StaticConstructObject_Internal` is the referencing function with the large stack frame (0x2B8 in 0.22.157). Its reference sits about 80 bytes after the prologue.
4. Take the shortest prefix (24 bytes or more) from the function start that is unique in the executable segments, and use it as the signature with `return MatchAddress`.

Script (needs numpy; runs in a few seconds on the 222 MB binary):

```python
#!/usr/bin/env python3
# find_sco.py <TheIsleServer-Linux-Shipping> - prints a UE4SS signature for StaticConstructObject_Internal
import re, struct, sys
import numpy as np

data = open(sys.argv[1], "rb").read()
phoff = struct.unpack_from("<Q", data, 32)[0]
phsz, phn = struct.unpack_from("<HH", data, 54)
segs = []
for i in range(phn):
    p_type, p_flags, off, va, _pa, filesz, _memsz, _al = struct.unpack_from("<IIQQQQQQ", data, phoff + i * phsz)
    if p_type == 1:
        segs.append((off, filesz, va, p_flags))
code = [(o, n, v) for o, n, v, f in segs if f & 1]
def off2va(o):
    for so, n, v, _f in segs:
        if so <= o < so + n: return v + (o - so)

s_off = data.find("UObject_StaticConstructObjectInternal".encode("utf-16-le") + b"\x00\x00")
s_va = off2va(s_off)
scope_objs = {struct.unpack_from("<I", data, m.start() - 4)[0]
              for m in re.finditer(re.escape(b"\xbe" + struct.pack("<I", s_va)), data) if data[m.start() - 5] == 0xBF}

found = []
for so, n, sva in code:
    seg = np.frombuffer(data, dtype=np.uint8, count=n, offset=so)
    disp = (seg[:-4].astype(np.int64) | (seg[1:-3].astype(np.int64) << 8) |
            (seg[2:-2].astype(np.int64) << 16) | (seg[3:-1].astype(np.int64) << 24))
    disp = np.where(disp >= 2 ** 31, disp - 2 ** 32, disp)
    rip_end = sva + np.arange(len(disp)) + 4
    for obj in scope_objs:
        for i in np.nonzero((rip_end + disp >= obj) & (rip_end + disp < obj + 0x40))[0]:
            o = so + int(i)
            if data[o - 1] & 0xC7 != 0x05: continue
            for back in range(4096):
                p = o - back
                if data[p - 1] in (0xCC, 0xC3) and data[p:p + 3] == b"\x55\x41\x57":
                    found.append(p); break

def frame_of(f):
    return struct.unpack_from("<I", data, f + 13)[0] if data[f + 10:f + 13] == b"\x48\x81\xec" else 0
found = sorted(set(found), key=lambda f: -frame_of(f))
for f in found:
    print(f"candidate va 0x{off2va(f):x} frame=0x{frame_of(f):x}  {data[f:f + 24].hex(' ')}")

f = found[0]
blob = b"".join(data[so:so + n] for so, n, _ in code)
for length in range(24, 64):
    if blob.count(data[f:f + length]) == 1:
        print('\nfunction Register()\n    return "%s"\nend\n\nfunction OnMatchFound(MatchAddress)\n    return MatchAddress\nend'
              % data[f:f + length].hex(" ").upper())
        break
```

Output on 0.22.157:

```
candidate va 0x4a274b0 frame=0x2b8  55 41 57 41 56 41 55 41 54 53 48 81 ec b8 02 00 00 49 89 fe 4c 8b 27 48
candidate va 0xc728f30 frame=0x0    55 41 57 41 56 41 55 41 54 53 48 83 ec 68 ...
candidate va 0x5309190 frame=0x0    55 41 57 41 56 41 55 41 54 53 48 83 ec 18 ...
```

Sanity checks before trusting a candidate: the stack frame is large (hundreds of bytes); the bytes after the prologue read the parameter struct (`mov r?, [rdi]`, `mov rax, [rdi+8]`, `mov r?d, [rdi+0x18]`, `mov rax, [rdi+0x28]`, `mov r?d, [rdi+0x70]`); and `StaticAllocateObject` (scope `UObject_StaticAllocateObject`, registered right after it by the same initialiser) is a nearby function a few KB later. After the swap, `UE4SS.log` must show `StaticConstructObject_Internal address: ... <- Lua Script` and `Installing detour`.

## Update checklist (Linux)

1. After any game update, open `TheIsle/Binaries/Linux/ue4ss/UE4SS.log` and search for `Fatal Error` and `Was unable to find AOB`. The game itself gives no sign that the mods are dead.
2. Copy the new `TheIsleServer-Linux-Shipping` somewhere readable and run `sigmatch.py`, then `find_sco.py` if `StaticConstructObject` reports `0 matches`.
3. Keep the previous signature files next to the new ones (for example `StaticConstructObject.lua.build24664709`). Steam replaces the old binary, so you cannot regenerate the old signatures later.
4. Replace only the broken file and restart the server. UE4SS reads signatures only at process start.
5. Regenerate your `.usmap` if you use one for offline tooling: the class layout changes with the patch. On this Linux build the Lua global `DumpUSMAP()`, called once from `ExecuteWithDelay` about 40 s after boot, writes `ue4ss/<Project>-<EngineVersion>-<SHA>.usmap` (2.97 MB on 0.22.157, Blueprint-generated types included).

## Related Linux gotchas

- The UE4SS Linux port loads mods from a lowercase `scripts/` folder (`Mods/<Mod>/scripts/main.lua`). `Scripts/` is silently ignored.
- The server process working directory is `TheIsle/Binaries/Linux`, so relative paths from Lua must start with `ue4ss/Mods/<Mod>/...`.
- A mod with an `enabled.txt` file loads even when `mods.txt` says `<Mod> : 0`. Delete `enabled.txt` to really disable it.
- UE4SS is injected with `LD_PRELOAD=<...>/ue4ss/libUE4SS.so` on the server start command. A game update does not touch the `ue4ss/` folder, so only the signatures need attention.
