# Source_Files/CSeries/cspixels.h

## File Purpose
Defines pixel type aliases and provides macros for converting between RGB color values and pixel representations at different bit depths (8-bit, 16-bit, 32-bit). This is a utility header for the Aleph One game engine's rendering subsystem.

## Core Responsibilities
- Define typedefs for pixel data at different color depths (`pixel8`, `pixel16`, `pixel32`)
- Provide macros to convert RGB color components (16-bit range) to packed pixel formats
- Provide macros to extract individual R, G, B components from packed pixel formats
- Define constants for color depth limits and component ranges

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `pixel8` | typedef | 8-bit indexed color pixel |
| `pixel16` | typedef | 16-bit RGB pixel (5-5-5 bit depth) |
| `pixel32` | typedef | 32-bit RGB pixel (8-8-8 bit depth) |

## Global / File-Static State
None.

## Key Functions / Methods
All meaningful content is macro-based; summarized below.

**Color Conversion Macros:**
- `RGBCOLOR_TO_PIXEL16(r, g, b)` ΓÇô Packs 16-bit RGB input (0x0000ΓÇô0xFFFF range) into 5-5-5 format (11 bits used: bits 15ΓÇô11=R, 10ΓÇô6=G, 5ΓÇô0=B). Returns `pixel16`.
- `RGBCOLOR_TO_PIXEL32(r, g, b)` ΓÇô Packs 16-bit RGB input into 8-8-8 format (bits 23ΓÇô16=R, 15ΓÇô8=G, 7ΓÇô0=B). Returns `pixel32`.

**Component Extractors (16-bit):**
- `RED16(p)`, `GREEN16(p)`, `BLUE16(p)` ΓÇô Extract 5-bit R, G, B components (output range 0x00ΓÇô0x1F).

**Component Extractors (32-bit):**
- `RED32(p)`, `GREEN32(p)`, `BLUE32(p)` ΓÇô Extract 8-bit R, G, B components (output range 0x00ΓÇô0xFF).

## Control Flow Notes
None; this is a pure utility header with no executable code or control flow.

## External Dependencies
- **`cstypes.h`** ΓÇô Provides integer typedefs (`uint8`, `uint16`, `uint32`) via SDL2 types.
- Guard macros prevent multiple inclusion.

**Notes:**
- Input range for converters is 16-bit (0x0000ΓÇô0xFFFF), but output bit depth is lower per format.
- The 16-bit format uses a 5-5-5 RGB packing (not the more common 5-6-5), leaving 1 unused bit.
- Extractors do *not* upscale; a 5-bit value stays 5-bit, requiring caller to scale if full 8-bit range is needed.
