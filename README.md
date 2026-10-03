# DQ8Recomp

An experimental native port of **Dragon Quest VIII: Journey of the Cursed King**
for PlayStation 2, built with [PS2Recomp](https://github.com/ran-j/PS2Recomp).
The game's MIPS code is translated to C++ locally and linked to a runtime that
implements the PS2 services it uses. Graphics use SDL3 GPU or a software reference
renderer.

**This is a development project, not a finished port.** The NTSC-U version
(`SLUS_212.07`) boots, loads saves, and reaches field gameplay, with sound from
the game's own drivers. Sustained playable performance and full-game
compatibility are still in progress. The PAL configuration is incomplete.

## About this fork

This is a fork of [Sinan-Karakaya/DQ8-Recompiled](https://github.com/Sinan-Karakaya/DQ8-Recompiled).
It uses a patched PS2Recomp,
[Yakuru-43/PS2Recomp](https://github.com/Yakuru-43/PS2Recomp/tree/perf/dq8-open-world-and-pad-record)
(branch `perf/dq8-open-world-and-pad-record`), which fixes the following:

| Symptom in game | Cause | Fix |
| --- | --- | --- |
| Dialogue close-ups stay black after fading out (Valentina's house, talking to King Trode in Farebury) | The stubs for `sin`, `cos`, `tan`, `atan`, `floor` and `fabs` read a float from `$f12`, but DQ8 calls the soft-float **double** versions (argument in 64-bit `$a0`, result in `$v0`). Event cameras got garbage angles and every object was culled. | Stubs use the EE soft-float double ABI |
| Characters' heads sit low and drift sideways ([issue #4](https://github.com/Sinan-Karakaya/DQ8-Recompiled/issues/4)) | The VU0 macro `VRSQRT` computed `1/sqrt(ft)` instead of `fs/sqrt(ft)`, breaking a distance-constraint helper | R5900/VU semantics for `VRSQRT`, `VDIV`, `VSQRT`, `RSQRT.S` and `DIV.S` (including ±MAX on division by zero) |
| Wrong VU1 results in programs using EFU instructions | The EFU opcode table was shifted by one from 0x77: ERSQRT ran as ESIN, ESIN as EATAN, EATAN as EEXP, and EEXP stopped the VU | Opcodes match the VU manual and PCSX2 |
| Matrix, normal and lighting maths wrong where the SDK is used | Several `libvu0` host stubs differed from the SDK code in the game: `MulMatrix` multiplied in reverse order, `Normalize`/`InnerProduct` included w, the `RotTransPers` flag was inverted | Stubs checked line by line against the original routines |

Also included:

- `DQ8_PAD_RECORD=<file>` records what you play (buttons and analog sticks) in
  the `DQ8_PAD_SCRIPT` format; replay it with `DQ8_PAD_SCRIPT=<file>`. Scripts
  are applied at the frame the game reads the pad, so a replay from the same
  save reaches the same place (useful for repeatable benchmarks). Scripts also
  accept `<frame> STICK <rx>,<ry>,<lx>,<ly> <hold>`.
- `DQ8_PAD_LIVE=<file>` feeds pad input to a running game: append lines in the
  `DQ8_PAD_SCRIPT` format, counted from the current frame.
- Open-world speed-ups: VU1 no longer clears a 64 KiB buffer at every program
  start (about 1 GB/s of memset), GS commands reach the render worker in
  batches, and `-DDQ8_RUNTIME_ARCH=native` optionally builds the runtime for
  your CPU. On the world map outside Farebury (Ryzen AI 7 350), the VU1 and GS
  changes took the game thread from 87% to 84% busy and rendering from 28.6 to
  29.3 frames/s (the game caps at 30); `native` gains about 1% more.
- `DQ8_GFX_SHOW_FPS=1` also logs completed renders per second.
- Opt-in traces: `DQ8_TRACE_VU1_RATE`, `PS2_VU_TRACE_CAMERA`, `DQ8_WATCH`.

On x86-64 Linux with recorded VU1 programs (see the
[Vector Units](https://github.com/Sinan-Karakaya/DQ8-Recompiled/wiki/Vector-Units)
page), the opening field holds the game's own cap of 30 renders/s. Without
them, the VU interpreter manages 10–15 frames/s.

After pulling these changes, run `git submodule update --init`, rebuild the
recompiler, run `setup.py recompile` again (the translator changed), then
rebuild the game.

**All of the changes in this fork were made by AI.** Claude, Anthropic's model,
running in Claude Code, did the investigation, code, commit messages and this
section, at the request of the repository owner, who tested the results in game.
Each commit message explains what was wrong, how it was found and how it was
fixed.

## Getting started

You need your own lawfully obtained game dump. This repository provides tools,
runtime code, and configuration; it does not provide the game, a BIOS, or a
prebuilt game executable.

```sh
git clone --recurse-submodules https://github.com/Yakuru-43/DQ8-Recompiled.git
cd DQ8-Recompiled
```

Follow the [build guide](https://github.com/Sinan-Karakaya/DQ8-Recompiled/wiki/Building)
to generate the game code and build the runtime. A default CMake build produces a
launcher stub, not the game.

- [Running and controls](https://github.com/Sinan-Karakaya/DQ8-Recompiled/wiki/Running)
- [Architecture](https://github.com/Sinan-Karakaya/DQ8-Recompiled/wiki/Architecture)
- [Tests and debugging](https://github.com/Sinan-Karakaya/DQ8-Recompiled/wiki/Testing)
- [Status and priorities](https://github.com/Sinan-Karakaya/DQ8-Recompiled/wiki/Project-Status)

## Contributing

See [CONTRIBUTE.md](CONTRIBUTE.md) for the workflow, testing expectations, and rules
for handling game data. Performance, compatibility, tooling, and documentation
contributions are welcome.

| Directory | Contents |
| --- | --- |
| `src/runtime` | Launcher, overlay dispatch, and game-specific integration |
| `src/gfx` | GS state, memory, trace replay, and SDL GPU renderer |
| `config` | Version-specific function maps and recompiler settings |
| `tools` | Extraction, analysis, code generation, and tests |
| `thirdparty` | Pinned PS2Recomp and SIMDe submodules |

Documentation is maintained in the
[wiki](https://github.com/Sinan-Karakaya/DQ8-Recompiled/wiki).

## License and attribution

Project code is available under [GNU GPL version 3](LICENSE). Dependencies retain
their own licenses; see [NOTICE](NOTICE).

This is an independent project, unaffiliated with Square Enix, Level-5, or Sony.
Dragon Quest and PlayStation names belong to their respective owners. The project
license grants no rights to the game or its assets.
