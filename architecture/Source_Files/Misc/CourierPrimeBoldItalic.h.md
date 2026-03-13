# Source_Files/Misc/CourierPrimeBoldItalic.h

## File Purpose
Binary TrueType/OpenType font file for "Courier Prime Bold Italic" converted to C header format. Contains complete glyph outlines, metrics, character mappings, and rendering hints required to display the font across all supported Unicode characters.

## Core Responsibilities
- **Character mapping**: Unicode codepoint to glyph index translation (cmap table)
- **Glyph geometry**: Outline contours and coordinate data for each character (glyf table)
- **Typography metrics**: Character widths, heights, advance widths (hmtx, hhea tables)
- **Font metadata**: Family name, style, copyright, version information (name table)
- **Rasterization hints**: Device-specific grid-fitting and hinting instructions (cvt, fpgm, prep, gasp tables)
- **Memory embedding**: Enables font inclusion in compiled binaries without external dependencies

## Key Types / Data Structures
| Component | Kind | Purpose |
|-----------|------|---------|
| Offset Table | Binary header | Directory with table locations and checksums |
| head | Table | Font bounding box, creation date, version |
| hhea | Table | Horizontal metrics (ascender, descender, line gap) |
| maxp | Table | Glyph count and maximum profile version |
| OS/2 | Table | Weight/width class, Panose classification |
| cmap | Table | Unicode ΓåÆ glyph index mappings |
| glyf | Table | Glyph contours and coordinate data |
| loca | Table | Byte offsets into glyf table |
| hmtx | Table | Per-glyph advance widths and side bearings |
| name | Table | Localized font names and copyright text |
| post | Table | PostScript information and glyph names |
| VDMX | Table | Vertical metrics for screen display |
| fpgm/prep | Tables | Font hinting programs |
| gasp | Table | Grid-fitting and anti-aliasing procedure |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| courier_prime_bold_italic | unsigned char[] | static | Complete binary font file (~11.5 KB); sole exported symbol |

## Key Functions / Methods
None. This is static binary font data with no executable code or function entry points.

## Control Flow Notes
This file contains no control flow. It is a data resource consumed by font rendering engines that:
1. Parse the Offset Table to locate individual tables
2. Read cmap table to map Unicode ΓåÆ glyph IDs
3. Fetch glyph outlines from glyf (indexed via loca)
4. Extract metrics from hmtx/hhea for text layout
5. Apply hinting instructions (fpgm/prep) during rasterization

## External Dependencies
- **OpenType/TrueType Specification**: Binary format follows official spec (big-endian, table-based structure)
- **Font Rasterization Engine**: Required to parse, hint, and render glyph outlines
- **Embedded License**: Contains OFL (Open Font License) text in name table
- **Character Encoding**: Supports extensive Unicode coverage via cmap subtables

**Notes:**
- Generated 2013-06-17 via `bin2h.py` conversion tool
- Monospace serif font designed for screenplay formatting
- Single file contains all glyphs for Bold Italic style
- Binary data is pure font metrics and geometry with no executable code
- Enables zero-dependency font embedding in compiled applications
