# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GFX-13 v1 (v1.1) is the pure assembly (TASM) version of the VGA Mode 13h graphics library, reconstructed from the C/inline assembly version (v2.1). It provides an 18-function API implemented entirely in 80x86 assembly with C-callable linkage. Two v2 functions (`GetImage` and `ScaleImage`) are not present in v1.

**Target platform:** Borland Turbo Assembler (TASM) 3.x + Borland C++ 3.1, DOS real mode, 80186+.

## Build

From the `test/` directory, run `build.bat`. Requires both `tasm` and `bcc` on PATH.

```bat
cd test
build.bat
```

The build script first assembles `gfx13.asm` with TASM, then compiles each test program with BCC and links against `gfx13.obj`. Output goes to `build.log`. To clean: `clean.bat`.

To assemble the library standalone (from `v1/`):

```
tasm /ml gfx13.asm
```

### Critical Build Flags

- **`/ml`** (TASM, case-sensitive symbols) - Required for correct C linkage. Without it, TASM uppercases all symbols and the linker cannot resolve `_SetMode13`, `_PutPixel`, etc.
- **`-1`** (BCC, 80186 mode) - Required because `SetPalette` uses `REP OUTSB`, a 186+ instruction.
- **`-P`** (BCC, force C++ mode) - Test programs are `.c` files compiled as C++.
- Always reassemble `gfx13.asm` and recompile all test programs after changes. Stale `.obj` files cause mysterious runtime failures.

## Architecture

### Assembly Conventions

- **`.MODEL SMALL, C`** - Small memory model with C calling convention. TASM automatically prepends underscores and generates standard stack frames for `PROC C` declarations.
- **`.186`** directive - Enables 80186+ instructions (e.g., `REP OUTSB` in `SetPalette`, `PUSH imm` in `FillTriangle`/`FillQuad`).
- Parameters are declared in `PROC C` lines (e.g., `Line PROC C x0:WORD, y0:WORD, ...`). TASM generates BP-relative access automatically.
- Local variables use `LOCAL` directives (e.g., `LOCAL textRows:WORD`).

### Macros

Three macros factor out repeated patterns (these were manually inlined in v2):

- **`FIXED_DIVIDE`** - Signed 16.16 fixed-point division: `(DX << 16) / CX`. Uses two-step unsigned 32/16 division with sign correction. Used by `FillTriangle` for DDA slope computation (4 expansions).
- **`CLIPPED_HFILL`** - Draws a clipped horizontal scanline fill. Clips against `clip_x1/y1/x2/y2`, computes pixel offset via the `y*256 + y*64 + x` shift pattern, and fills with `REP STOSB`. Used by `FillTriangle` (4 expansions).
- **`DDA_ADVANCE`** - Advances left/right 16.16 fixed-point DDA accumulators by their slopes. Used by `FillTriangle` for scanline edge walking.

### Label Naming

Functions use `FUNCTIONNAME_LABEL` global labels (e.g., `LINE_DRAW`, `PUTPIXEL_NOCLIP`, `SETTEXTMODE_DONE`). The `Line` function is the largest (~430 lines) and uses the most labels for its Cohen-Sutherland clipping state machine.

### Key Differences from v2

- v2 uses BCC inline `asm` blocks inside C functions; v1 is pure TASM with `PROC C` directives.
- v2 suffers from BCC's silent Jcc displacement truncation bug (see parent CLAUDE.md). v1 uses TASM which correctly *reports* the error instead of silently truncating, but `.186` mode still only supports SHORT (±127 byte) conditional jumps. The `JUMPS` directive must be used to have TASM automatically generate inverted-Jcc + near-JMP trampolines for out-of-range targets.
- v1 factors repeated assembly patterns into TASM macros (`FIXED_DIVIDE`, `CLIPPED_HFILL`, `DDA_ADVANCE`); v2 duplicates these inline at each use site.
- v1 has 18 functions; v2 has 20. The two v2-only functions are `GetImage` (copies a screen region to a buffer) and `ScaleImage` (nearest-neighbor scaled blit with clipping).

## Lessons Learned

### Assembly and TASM

- **`JUMPS` directive is required.** With `.186`, conditional jumps (Jcc) only support SHORT displacement (±127 bytes). Large functions like `Line` (~430 lines) and functions with macro expansions (`FillTriangle`, `FillRectangle`) easily exceed this limit. The `JUMPS` directive tells TASM to automatically generate inverted-Jcc + near-JMP trampolines for out-of-range targets. Without it, TASM reports "Relative jump out of range" errors. Always include `JUMPS` after the `.186` directive.
- **Macro `LOCAL` must immediately follow `MACRO`.** In TASM, the `LOCAL` directive inside a macro must be the very first statement after the `MACRO` line — no blank lines, comments, or other directives can intervene. If a blank line separates `MACRO` from `LOCAL`, TASM does not recognize the `LOCAL` as belonging to the macro. Labels become global instead of per-expansion unique, causing "Expecting pointer type" (unresolved forward references for jump targets) and "Symbol already different kind" (label collisions on second and subsequent expansions).
- **Do not declare `PUBLIC` symbols without implementations.** Every `PUBLIC` declaration must have a corresponding `PROC` in the code segment. TASM reports "Undefined symbol" for any `PUBLIC` name that has no matching label or procedure.
- **Memory model SMALL is correct for this library.** The API passes segment addresses as explicit WORD parameters (e.g., `0xA000` for VGA memory) and manually sets ES via `MOV ES, segment_value`. Far memory access is handled in the assembly code, not through the compiler's memory model. No language-level far pointers are used.

### Code Changes

- When adding or modifying a `PROC`, ensure its name appears in the `PUBLIC` declarations block at the top of `gfx13.asm` and in `gfx13.h`.
- When changing a function signature, update the `PROC C` parameter list, the header prototype, and every call site across all 8 test programs.
- The `gfx13.asm` file is ~2100 lines. When reordering or editing functions, verify the total line count and confirm no `PROC`/`ENDP` pairs were accidentally deleted.
- `STOSB` implicitly modifies ES:DI. Always `PUSH ES` / `POP ES` around segments that set ES for VGA writes.
