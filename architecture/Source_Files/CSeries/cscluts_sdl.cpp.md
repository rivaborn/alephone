# Source_Files/CSeries/cscluts_sdl.cpp

## File Purpose
SDL-based implementation of CLUT (Color Look-Up Table) resource handling for the Aleph One game engine. Converts classic Mac CLUT resource format to engine color tables and defines global color constants used throughout the engine.

## Core Responsibilities
- Define global color constants (`rgb_black`, `rgb_white`, `system_colors`)
- Parse Mac binary CLUT resource format from memory
- Convert 16-bit big-endian color components to engine color table format
- Manage endian conversion for cross-platform compatibility

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `RGBColor` | struct | 16-bit per-channel color (red, green, blue); defined in cseries.h |
| `color_table` | struct | Not defined in this file; inferred to contain `color_count` and `colors[]` array |
| `LoadedResource` | class | RAII wrapper for in-memory resource data; defined in FileHandler.h |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rgb_black` | `RGBColor` | global | Constant black color (0x0000 all channels) |
| `rgb_white` | `RGBColor` | global | Constant white color (0xffff all channels) |
| `system_colors` | `RGBColor[NUM_SYSTEM_COLORS]` | global | Predefined system UI colors (dark/light gray) |

## Key Functions / Methods

### build_color_table
- **Signature:** `void build_color_table(color_table *table, LoadedResource &clut)`
- **Purpose:** Parse Mac resource CLUT format and populate engine color table structure.
- **Inputs:** 
  - `table`: pointer to destination color table (output parameter)
  - `clut`: LoadedResource containing raw CLUT binary data from Mac resource fork
- **Outputs/Return:** None (writes to `*table`)
- **Side effects:** 
  - Modifies `table->color_count` and `table->colors[]`
  - Allocates and uses SDL_RWops stream (closed before return)
- **Calls (direct):** 
  - `clut.GetPointer()`, `clut.GetLength()` (LoadedResource)
  - `SDL_RWFromMem()`, `SDL_RWseek()`, `SDL_ReadBE16()`, `SDL_RWclose()` (SDL2)
  - `std::min()` (standard library)
- **Notes:** 
  - Assumes CLUT resource format: 6-byte header, then count field at offset 6 (big-endian uint16), then color triplets (skip 2 bytes per color, then R/G/B as 16-bit BE)
  - Clamps color count to 256 maximum
  - Uses `assert(p)` for stream creation validation; no error handling if assertion fails

## Control Flow Notes
Part of engine initialization/resource loading pipeline. Called when game loads CLUT resources from Mac data forks (e.g., during map/scenario loading). Global color constants are initialized statically at module load time and used throughout UI/rendering systems.

## External Dependencies
- **SDL2**: `SDL_RWops`, `SDL_RWFromMem()`, `SDL_RWseek()`, `SDL_ReadBE16()`, `SDL_RWclose()` ΓÇö memory-based I/O and big-endian integer reading
- **FileHandler.h**: `LoadedResource` class for resource management
- **cseries.h**: `RGBColor` struct definition; pulls in platform-specific macros and SDL headers
- **Implicit dependencies**: `color_table` structure definition, `NUM_SYSTEM_COLORS` constant (defined elsewhere)
