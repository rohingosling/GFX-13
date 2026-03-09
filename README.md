# GFX-13

A VGA Mode 13h (320x200, 256 colors) graphics library for DOS, providing 2D drawing primitives, image blitting, and palette manipulation.

Originally written in 1991 for use in DOS demos and games. The original source was lost; both versions here were reconstructed from surviving build artifacts.

## Versions

| Version | Language | Functions | Toolchain |
|---------|----------|-----------|-----------|
| **v1** (v1.1) | Pure 80x86 assembly (TASM) | 18 | TASM 3.x + BCC 3.1 |
| **v2** (v2.1) | C with inline assembly | 20 | BCC 3.1 |

v2 adds `GetImage` and `ScaleImage` on top of v1's API.

## API

```c
// Video Mode
void SetMode13   (void);                      // Enter 320x200, 256-color mode
void SetTextMode (BYTE rows);                 // Return to text mode (25, 43, or 50 rows)
BYTE GetTextMode (void);                      // Query current text mode row count

// Palette and Clipping
void SetPalette  (BYTE col, WORD count, WORD segment, WORD dataOffset);
void SetClipping (int x0, int y0, int x1, int y1);

// Screen Buffer
void ClearScreen (BYTE col, WORD dest);       // Fill screen with a single color
void FlipScreen  (WORD source, WORD dest);    // Copy entire screen buffer
void WaitRetrace (void);                      // Sync with vertical retrace

// Pixels
void PutPixel    (WORD x, WORD y, BYTE col, BYTE clip, WORD dest);
BYTE GetPixel    (WORD x, WORD y, BYTE clip, WORD source);

// Line and Outline Primitives
void Line        (WORD x0, WORD y0, WORD x1, WORD y1, BYTE col, BYTE clip, WORD dest);
void Rectangle   (WORD x0, WORD y0, WORD x1, WORD y1, BYTE col, WORD dest);
void Triangle    (WORD x0, WORD y0, WORD x1, WORD y1, WORD x2, WORD y2, BYTE col, WORD dest);
void Quad        (WORD x0, WORD y0, WORD x1, WORD y1, WORD x2, WORD y2, WORD x3, WORD y3, BYTE col, WORD dest);

// Filled Primitives
void FillRectangle (WORD x0, WORD y0, WORD x1, WORD y1, BYTE col, WORD dest);
void FillTriangle  (int x0, int y0, int x1, int y1, int x2, int y2, BYTE col, WORD dest);
void FillQuad      (int x0, int y0, int x1, int y1, int x2, int y2, int x3, int y3, BYTE col, WORD dest);

// Image Blitting
void PutImage    (WORD x, WORD y, WORD xs, WORD size, BYTE mask, WORD source_seg, WORD source_offs, WORD dest);
void GetImage    (WORD x0, WORD y0, WORD x1, WORD y1, WORD source, WORD dest_seg, WORD dest_offs);         // v2 only
void ScaleImage  (WORD x0, WORD y0, WORD x1, WORD y1, WORD source_xs, WORD source_ys, BYTE mask,           // v2 only
                  WORD source_seg, WORD source_offs, WORD dest);
```

All `dest` and `source` parameters are 16-bit segment addresses (e.g., `0xA000` for VGA video memory).

## Usage

```c
#include "gfx13.h"

#define VGA 0xA000

int main(void)
{
    SetMode13();

    ClearScreen(0, VGA);
    FillTriangle(160, 10, 10, 190, 310, 190, 4, VGA);
    Line(0, 0, 319, 199, 15, 1, VGA);
    PutPixel(160, 100, 14, 1, VGA);

    getch();
    SetTextMode(25);
    return 0;
}
```

## Build

Requires [DOSBox](https://www.dosbox.com/) (or real DOS) with Borland Turbo Assembler (TASM) 3.x and Borland C++ 3.1 on PATH.

### v1 (Assembly)

```bat
cd v1\test
build.bat
```

Assembles `gfx13.asm` with TASM (`/ml` for case-sensitive C linkage), then compiles and links 8 test programs with BCC.

### v2 (C + Inline Assembly)

```bat
cd v2\test
build.bat
```

Compiles `gfx13.c` and links 8 test programs with BCC (`-1` for 80186 mode, `-P` for C++ mode).

### Clean

Run `clean.bat` from either `test/` directory to remove build artifacts.

## Test Suite

Both versions include 8 visual test programs:

| Test | Description |
|------|-------------|
| `testpix`  | Pixel drawing, reading, and screen clearing |
| `testline` | Line drawing with Bresenham's algorithm |
| `testrect` | Outline and filled rectangles |
| `testtri`  | Outline and filled triangles |
| `testquad` | Outline and filled quadrilaterals |
| `testimg`  | Image blitting with `PutImage` |
| `testpal`  | VGA palette manipulation |
| `testclip` | Clipping rectangle tests |

Each test program displays multiple screens. Press any key to advance between screens.

## Repository Structure

```
gfx-13/
├── v1/                     Pure TASM assembly (v1.1, 18 functions)
│   ├── gfx13.asm           Library source (~2100 lines)
│   ├── gfx13.h             C-callable header
│   └── test/               8 test programs + build scripts
├── v2/                     C + inline assembly (v2.1, 20 functions)
│   ├── gfx13.c             Library source (~2700 lines)
│   ├── gfx13.h             C-callable header
│   └── test/               8 test programs + build scripts
└── CLAUDE.md               Project guidance for Claude Code
```

## Technical Details

- **Target:** 80186+ real mode, small memory model
- **Resolution:** 320x200, 256 colors (VGA Mode 13h)
- **Pixel offset:** `y * 320 + x` computed via shifts as `y*256 + y*64 + x`
- **Line clipping:** Cohen-Sutherland algorithm
- **Filled polygons:** Scanline rasterization with 16.16 fixed-point DDA edge walking
- **Palette streaming:** `REP OUTSB` to VGA DAC (requires 80186+)

## License

This project is provided as-is for educational and archival purposes.
