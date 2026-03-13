# Source_Files/CSeries/csfonts.h

## File Purpose
Header defining font styling constants and the TextSpec structure for the Aleph One game engine. Provides bitflags for text styling (bold, italic, underline, shadow) and a unified data structure to encapsulate font metadata and file paths.

## Core Responsibilities
- Define text style constants as bitflags for composable text styling
- Provide TextSpec struct to bundle font configuration (ID, size, style, height adjustment, and font file paths)
- Support TTF font variants (normal, oblique, bold, bold_oblique) for style rendering
- Enable consistent font specification across the engine

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| TextSpec | struct | Encapsulates font ID, style flags, size, height adjustment, and paths to four font file variants |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| styleNormal | const int | global | Bitflag (0) for unstyled text |
| styleBold | const int | global | Bitflag (1) for bold styling |
| styleItalic | const int | global | Bitflag (2) for italic styling |
| styleUnderline | const int | global | Bitflag (4) for underlined text |
| styleShadow | const int | global | Bitflag (16) for drop shadow effect |

## Key Functions / Methods
None (header-only definitions).

## Control Flow Notes
Not inferable from this file. This is a data structure definition; font loading, caching, and rendering occur in implementation files that include this header.

## External Dependencies
- `cstypes.h` ΓÇô provides int16, uint16 typedefs (SDL2/SDL_types.h-based)
- `<string>` ΓÇô STL for font file path storage

## Notes
- Style constants are powers of 2, enabling bitwise OR composition (e.g., `styleBold | styleItalic`)
- styleOutline (8) is commented out as incompatible with TTF rendering
- Four font path fields (normal, oblique, bold, bold_oblique) avoid runtime font synthesis; variants are preselected at load time
