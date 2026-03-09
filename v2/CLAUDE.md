# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A VGA Mode 13h (320x200, 256 colors) graphics library for DOS, written in C with Borland Turbo C++ inline assembly. The original `gfx13.c` source was lost; the current file was reconstructed from the surviving header (`gfx13.h`) and compiled object (`gfx13.obj`) recovered from the Balls v1.1 demo.

**Target platform:** Borland Turbo C++ 3.1, DOS real mode, 486 (no FPU).

**Compiler note:** Although this is a C library, it is compiled with Borland C++ 3.1 (`bcc`). The `-P` flag forces C++ compilation mode; the `extern "C"` guards in `gfx13.h` ensure correct C linkage.

## Build

This is a library (no `main`), compiled with Borland C++ 3.1. There is no makefile in this directory; it is compiled as part of a larger project that includes it. To compile the object file standalone:

```
bcc -1 -P -I. -c gfx13.c
```

The `-1` flag enables 80186 code generation (required for `REP OUTSB`). The `-P` flag forces C++ compilation mode.

## Architecture

Two files, no external dependencies beyond Turbo C++ 3.1 and BIOS/VGA hardware:

- **`gfx13.h`** - Public API (20 functions). Types: `BYTE` = `unsigned char`, `WORD` = `unsigned` (16-bit).
- **`gfx13.c`** - All implementations in Turbo C inline `asm` blocks targeting 16-bit real mode.

### Key Conventions

- **Segment addressing:** All `dest` and `source` parameters are 16-bit segment addresses (e.g., `0xA000` for VGA video memory). Offsets default to zero (start of segment) except in `PutImage`, `GetImage`, `ScaleImage`, and `SetPalette` where explicit `segment:offset` pairs are used.
- **y * 320 + x via shifts:** The linear offset computation `y * 320 + x` is done as `XCHG CH,CL; ADD BX,CX; SHR CX,2; ADD BX,CX` (i.e., `y*256 + y*64 + x`). This pattern recurs in nearly every function.
- **Clipping:** File-scope static variables `clip_x1/y1/x2/y2` (default 0,0,319,199). Set via `SetClipping()`. Used by `FillRectangle`, `FillTriangle`, `FillQuad`, and `ScaleImage` (inline clipped horizontal fills). `PutPixel`, `GetPixel`, and `Line` clip when their `clip` parameter is nonzero. `Line` uses Cohen-Sutherland; `PutPixel` and `GetPixel` use a simple bounds test. Outline primitives (`Rectangle`, `Triangle`, `Quad`) delegate to `Line` with `clip = 1`.
- **Label scoping:** Turbo C++ inline assembly uses C-style labeled blocks (`LABEL: asm { ... }`) for branching between asm blocks within a function.

### Function Groups

| Group              | Functions                                               |
|--------------------|---------------------------------------------------------|
| Video mode         | `SetMode13`, `SetTextMode`, `GetTextMode`               |
| Pixel              | `PutPixel`, `GetPixel`                                  |
| Image blit         | `PutImage`, `GetImage`, `ScaleImage`                    |
| Screen buffer      | `FlipScreen`, `ClearScreen`                             |
| Palette / sync     | `SetPalette`, `WaitRetrace`                             |
| Clipping           | `SetClipping`                                           |
| Line               | `Line`                                                  |
| Rectangle          | `Rectangle` (outline), `FillRectangle`                  |
| Triangle           | `Triangle` (outline), `FillTriangle`                    |
| Quad               | `Quad` (outline), `FillQuad`                            |

### Filled Polygon Rendering

`FillTriangle` and `FillQuad` use scanline rasterization with 16.16 fixed-point edge walking. Vertices are sorted by Y, edges are walked with DDA accumulators, and each scanline is drawn via an inlined clipped horizontal fill (X+Y clip, sort, clamp, REP STOSB). `FillQuad` splits the quad into two triangles and fills each half.

### ScaleImage

Uses 16.16 fixed-point DDA for nearest-neighbor scaling with full rectangle clipping. DDA accumulators are pre-adjusted for clipped edges so the source-to-destination mapping remains correct.

## Lessons Learned

### Build

- The build script (`test/build.bat`) uses `bcc` (Borland C++) with `-P` (force C++ mode), not `tcc` with `-p-`. Keep CLAUDE.md and the build script consistent when either changes.
- The `-1` flag (80186 code generation) is **required** because `SetPalette` uses `REP OUTSB`, a 186+ instruction. Do not remove this flag.
- Always rebuild both `gfx13.c` and the test programs after changes. Stale `.obj` files are a common source of mysterious runtime failures.

### Inline Assembly

- **BCC's inline assembler silently truncates conditional jump (Jcc) displacements that exceed ±127 bytes in 186 mode (`-1` flag).** This was confirmed as the root cause of the Line function hang, and the same issue was found in FillTriangle and ScaleImage. The fix is to use inverted-short-Jcc + near-JMP trampolines (`JNZ SHORT_LABEL; JMP FAR_LABEL; SHORT_LABEL:`), or move the branch decision into C code (`if (!clip) goto LABEL;`). Unconditional JMP is always near and safe at any distance. **Any function with large asm blocks (>60 instructions between a Jcc and its target) must be audited for this issue.**
- When inlining code (e.g., replacing `HorizontalLine` calls with inline fills in `FillTriangle`), use unique label prefixes per call site (e.g., `FTD1_`, `FTFB_`, `FTD2_`, `FTFT_`) to avoid label collisions.
- `STOSB` implicitly modifies DI. Always `PUSH ES` / `POP ES` around segments that set ES for VGA writes, and be aware that DI is clobbered.

### Code Changes

- When changing a function signature (e.g., adding a parameter to `SetTextMode`), update every call site across all test programs before rebuilding.
- When removing a public API function (e.g., `HorizontalLine`), update: the header, all internal callers, all test programs, the CLAUDE.md function table, and any comments that reference it.
- When reordering functions in a large file, verify the output line count matches the original. Missing functions are easy to overlook in a file with 2500+ lines.

### Debugging

- If a test program hangs or crashes after a code change that should be benign (e.g., changing `clip=0` to `clip=TRUE` for coordinates already within bounds), suspect build artifacts or compiler code generation issues before spending time on logic analysis.
- Test each screen in isolation by commenting out the others to narrow down which code path fails.
