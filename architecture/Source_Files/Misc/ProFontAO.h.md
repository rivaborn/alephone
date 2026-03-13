# Source_Files/Misc/ProFontAO.h

## File Purpose
Embeds the ProFont monospace font as static binary data in a C header file for runtime font loading. Eliminates external font file dependencies by compiling the font directly into the executable.

## Core Responsibilities
- Provides raw TrueType/OpenType font binary data as a static array
- Exports font size constant for runtime access
- Enables font rendering without filesystem dependencies
- Generated via automated binary-to-header conversion

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `pro_font_ao` | `unsigned char[]` | global | Embedded font binary data (45,896 bytes) |
| `pro_font_ao_size` | `unsigned int` | global | Size constant of font data in bytes |

## Key Functions / Methods
None.

## Control Flow Notes
Not applicableΓÇöthis file contains only static data. The font data is linked at compile-time and accessed by graphics/rendering systems at runtime via the global array and size constant.

## External Dependencies
- Generated via `/usr/local/bin/bin2h.py` (build-time tool)
- No runtime dependencies
- Intended for inclusion in game engine or UI rendering code that consumes TrueType font data

**Notes**: The binary data is hexadecimal-encoded as `unsigned char` values. The font identifier "ProFontAO" suggests the Adobe Original variant of ProFont, a popular monospace programming font. At 45.8 KB, this is a substantial embedded asset, likely used for in-game UI text rendering.
