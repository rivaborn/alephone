# Source_Files/CSeries/cscluts.h

## File Purpose
Defines color lookup table (CLUT) structures and color management for the Aleph One game engine. Provides data structures for representing RGB colors and palettes, along with a function to load color tables from resources and a global system color registry.

## Core Responsibilities
- Define RGB color representation (16-bit per channel)
- Define indexed color table structure (up to 256 colors)
- Provide resource loading for color tables
- Maintain global system color constants (black, white, system palette)
- Support color palette management for rendering

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `rgb_color` | struct | Single RGB color with 16-bit red, green, blue channels |
| `color_table` | struct | Indexed palette; contains color count (Γëñ256) and color array |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rgb_black` | RGBColor | extern | Predefined black color constant |
| `rgb_white` | RGBColor | extern | Predefined white color constant |
| `system_colors` | RGBColor[2] | extern | System palette (2 colors: gray15Percent, windowHighlight) |

## Key Functions / Methods

### build_color_table
- Signature: `void build_color_table(color_table *table, LoadedResource &clut)`
- Purpose: Load/initialize a color table from a LoadedResource (likely from a resource file)
- Inputs: Pointer to color_table struct to populate; reference to LoadedResource containing palette data
- Outputs/Return: None (modifies table in-place)
- Side effects: Modifies the color_table structure; likely allocates or initializes memory
- Calls: Not inferable from this file
- Notes: Implementation defined elsewhere; typical usage during engine initialization

## Control Flow Notes
Color tables are loaded during initialization/resource-loading phase (not per-frame). System colors are global constants available throughout the engine lifetime.

## External Dependencies
- `cstypes.h` ΓÇô for `uint16`, `uint32` integer type definitions
- `LoadedResource` ΓÇô forward declared; defined elsewhere; used for resource loading
- `RGBColor` ΓÇô forward declared; defined elsewhere; likely a struct or typedef for color representation
