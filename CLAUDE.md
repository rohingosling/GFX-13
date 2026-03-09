# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GFX-13 is a VGA Mode 13h (320x200, 256 colors) graphics library for DOS. Two implementations exist:

- **v1** (v1.1) — Pure 80x86 assembly (TASM), 18 functions. Reconstructed from the C/inline assembly version (v2).
- **v2** (v2.1) — C with Borland Turbo C++ inline assembly, 20 functions. Reconstructed from the surviving header (`gfx13.h`) and compiled object (`gfx13.obj`).

**Target platform:** Borland Turbo Assembler (TASM) 3.x + Borland Turbo C++ 3.1, DOS real mode, 80186+.

## Repository Structure

- **`v1/`** - Pure assembly version (v1.1, TASM). 18-function API (v2 minus `GetImage` and `ScaleImage`).
  - `gfx13.asm` - All implementations in pure TASM assembly with `PROC C` directives.
  - `gfx13.h` - Public API header (18 functions). Types: `BYTE` = `unsigned char`, `WORD` = `unsigned` (16-bit).
  - `test/` - Same 8 test programs as v2, compiled with BCC and linked against the TASM-assembled `gfx13.obj`.
- **`v2/`** - C/inline assembly version (v2.1). 20-function API.
  - `gfx13.h` - Public API header (20 functions). Types: `BYTE` = `unsigned char`, `WORD` = `unsigned` (16-bit).
  - `gfx13.c` - All implementations using Turbo C inline `asm` blocks targeting 16-bit real mode.
  - `test/` - 8 visual test programs (testpix, testline, testrect, testtri, testquad, testimg, testpal, testclip), each with `getch()` between test screens.
  - `test/testutil.h` - Shared header with CP437 box-drawing project banner (static functions, no linker conflicts).
- **`Backup/`** - Backup snapshots of v2.

## Build

### v1 (TASM + BCC)

From `v1/test/`, run `build.bat`. Requires both `tasm` and `bcc` on PATH.

```bat
cd v1\test
build.bat
```

The build script assembles `gfx13.asm` with TASM, then compiles each test program with BCC and links against `gfx13.obj`. To assemble standalone (from `v1/`): `tasm /ml gfx13.asm`.

### v2 (BCC only)

From `v2/test/`, run `build.bat`. Requires `bcc` on PATH.

```bat
cd v2\test
build.bat
```

The build script compiles `gfx13.c` then links each test program against `gfx13.obj`. To compile standalone (from `v2/`): `bcc -1 -P -I. -c gfx13.c`.

### Shared

Output goes to `build.log`. To clean: `clean.bat` (deletes `.exe`, `.obj`, `build.log`).

### Critical Build Flags

- **`/ml`** (TASM, case-sensitive symbols) - Required for correct C linkage. Without it, TASM uppercases all symbols.
- **`-1`** (BCC, 80186 mode) - Required because `SetPalette` uses `REP OUTSB`, a 186+ instruction. Do not remove.
- **`-P`** (BCC, force C++ mode) - The build script uses `bcc -P`, not `tcc -p-`. Keep CLAUDE.md and build script consistent when either changes.
- Always rebuild all source files and test programs after changes. Stale `.obj` files cause mysterious runtime failures.

## Architecture

### Segment Addressing

All `dest` and `source` parameters are 16-bit segment addresses (e.g., `0xA000` for VGA video memory). Offsets default to zero except in `PutImage`, `SetPalette`, and (v2 only) `GetImage` and `ScaleImage`, where explicit `segment:offset` pairs are used.

### Pixel Offset Calculation

The linear offset `y * 320 + x` is computed via shifts: `y*256 + y*64 + x` (`XCHG CH,CL; ADD BX,CX; SHR CX,2; ADD BX,CX`). This pattern recurs in nearly every function.

### Clipping

File-scope static variables `clip_x1/y1/x2/y2` (default 0,0,319,199), set via `SetClipping()`. Filled primitives (`FillRectangle`, `FillTriangle`, `FillQuad`, and v2's `ScaleImage`) clip inline. `PutPixel`, `GetPixel`, and `Line` clip when their `clip` parameter is nonzero. `Line` uses Cohen-Sutherland. Outline primitives (`Rectangle`, `Triangle`, `Quad`) delegate to `Line` with `clip = 1`.

### Filled Polygon Rendering

`FillTriangle` and `FillQuad` use scanline rasterization with 16.16 fixed-point edge walking. Vertices are sorted by Y, edges are walked with DDA accumulators, and each scanline is drawn via an inlined clipped horizontal fill. `FillQuad` splits the quad into two triangles.

### ScaleImage (v2 only)

Uses 16.16 fixed-point DDA for nearest-neighbor scaling with full rectangle clipping. DDA accumulators are pre-adjusted for clipped edges. Not present in v1.

### Function Groups

| Group              | v1 + v2                                                 | v2 only              |
|--------------------|---------------------------------------------------------|----------------------|
| Video mode         | `SetMode13`, `SetTextMode`, `GetTextMode`               |                      |
| Pixel              | `PutPixel`, `GetPixel`                                  |                      |
| Image blit         | `PutImage`                                              | `GetImage`, `ScaleImage` |
| Screen buffer      | `FlipScreen`, `ClearScreen`                             |                      |
| Palette / sync     | `SetPalette`, `WaitRetrace`                             |                      |
| Clipping           | `SetClipping`                                           |                      |
| Line               | `Line`                                                  |                      |
| Rectangle          | `Rectangle` (outline), `FillRectangle`                  |                      |
| Triangle           | `Triangle` (outline), `FillTriangle`                    |                      |
| Quad               | `Quad` (outline), `FillQuad`                            |                      |

## Lessons Learned

### TASM (v1)

- **`JUMPS` directive is required.** With `.186`, conditional jumps (Jcc) only support SHORT displacement (±127 bytes). Large functions like `Line` and functions with macro expansions (`FillTriangle`) easily exceed this limit. `JUMPS` tells TASM to automatically generate inverted-Jcc + near-JMP trampolines. Always include `JUMPS` after the `.186` directive.
- **Macro `LOCAL` must immediately follow `MACRO`.** No blank lines, comments, or other directives can intervene. Violating this causes label collisions on macro re-expansion.
- **Do not declare `PUBLIC` symbols without implementations.** TASM reports "Undefined symbol" for any `PUBLIC` name that has no matching `PROC`.

### Inline Assembly (v2)

- **BCC's inline assembler silently truncates conditional jump (Jcc) displacements that exceed ±127 bytes in 186 mode (`-1` flag).** This caused hangs in Line, FillTriangle, and ScaleImage. The fix is inverted-short-Jcc + near-JMP trampolines (`JNZ SHORT_LABEL; JMP FAR_LABEL; SHORT_LABEL:`), or moving the branch decision into C code (`if (!clip) goto LABEL;`). Unconditional JMP is always near and safe. **Any function with large asm blocks (>60 instructions between a Jcc and its target) must be audited for this issue.**
- When inlining code, use unique label prefixes per call site (e.g., `FTD1_`, `FTFB_`) to avoid label collisions.
- `STOSB` implicitly modifies DI. Always `PUSH ES` / `POP ES` around segments that set ES for VGA writes, and be aware that DI is clobbered.

### Code Changes

- When changing a function signature, update every call site across all test programs before rebuilding.
- When removing a public API function, update: the header, all internal callers, all test programs, the CLAUDE.md function table, and any comments that reference it.
- When reordering functions in a large file, verify the output line count matches the original. Missing functions are easy to overlook in a 2500+ line file.

### Debugging

- If a test program hangs after a seemingly benign change, suspect build artifacts or compiler code generation issues before logic analysis.
- Test each screen in isolation by commenting out the others to narrow down which code path fails.
