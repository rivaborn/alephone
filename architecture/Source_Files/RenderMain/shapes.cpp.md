# Source_Files/RenderMain/shapes.cpp

## File Purpose
Manages loading and runtime access to game sprite collections (shapes), including bitmap data, color tables, shading tables, and special effects like infravision tinting. Converts compressed shape data from disk into renderable SDL surfaces and maintains lookup tables for light levels and visual effects.

## Core Responsibilities
- Load shape collections from disk (Marathon 1 & 2 formats) and parse metadata, bitmaps, color tables, and animation definitions
- Build and maintain shading tables for multiple bit depths (8/16/32-bit), supporting dynamic darkness/lighting
- Handle RLE (run-length encoded) bitmap decompression and coordinate transformations (mirroring)
- Convert shape data to SDL surfaces with proper color palettes and transparency
- Manage infravision tinting system for light-enhancement goggles effect
- Support runtime shape patching (replacing/modifying shapes after initial load)
- Provide accessor API for other systems to retrieve shape data by collection and index

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `collection_header` | struct | Runtime state of a loaded collection: shading tables, status flags, pointer to definition |
| `collection_definition` | struct | Metadata for a shape collection: counts, offsets, color tables, shape arrays |
| `bitmap_definition` | struct | Single bitmap properties: dimensions, bytes per row, flags, RLE vs raw storage |
| `high_level_shape_definition` | struct | Animated sprite: view count, frames per view, sounds, animation indices |
| `low_level_shape_definition` | struct | Individual sprite frame: flags, light intensity, bitmap index, world bounds |
| `rgb_color_value` | struct | Color: R/G/B components, flags (self-luminescent), palette index |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `collection_headers[]` | `collection_header[MAXIMUM_COLLECTIONS]` | static | Array of all loaded shape collections |
| `ShapesFile` | `OpenedFile` | static | Currently open M2 shapes file handle |
| `M1ShapesFile` | `OpenedResourceFile` | static | Marathon 1 resource fork file |
| `shapes_file_version` | enum | static | Indicates M1_SHAPES_VERSION or M2_SHAPES_VERSION |
| `global_shading_table16/32` | `pixel16*/pixel32*` | static | Global per-component shading lookup tables |
| `number_of_shading_tables` | short | static | Bit-depth dependent (32, 64, or 256) |
| `shading_table_size` | short | static | Bytes per shading table |
| `shapes_patch` | `std::vector<uint8>` | static | Runtime patch data applied over shapes |
| `CollectionTints[]` | `short[NUMBER_OF_COLLECTIONS]` | static | Infravision tint color index per collection |
| `tint_colors16[]`, `tint_colors8[]` | color arrays | static | Color definitions for infravision effect |

## Key Functions / Methods

### get_shape_surface
- **Signature:** `SDL_Surface *get_shape_surface(int shape, int inCollection = NONE, byte** outPointerToPixelData = NULL, float inIllumination = -1.0f, bool inShrinkImage = false)`
- **Purpose:** Convert a shape descriptor (or collection + index) to an SDL surface for rendering, optionally with custom illumination level or shrinking.
- **Inputs:** Shape descriptor or (collection, low-level index); illumination factor [0.0ΓÇô1.0] or negative for default; shrink flag
- **Outputs/Return:** SDL_Surface (palette + pixel data); optional pointer to separately allocated RLE pixel buffer
- **Side effects:** Allocates SDL surface and optionally malloc'd pixel buffer (caller must free); extracts color palette from CLUT or shading table
- **Calls:** `extended_get_shape_bitmap_and_shading_table()`, `get_bitmap_definition()`, `get_collection_colors()`, `get_collection_definition()`; SDL color/surface functions
- **Notes:** RLE shapes may allocate extra pixel storage; row-major vs. column-major layout and mirroring flags handled; shrinking uses nearest-neighbor sampling

### load_collection
- **Signature:** `static bool load_collection(short collection_index, bool strip)`
- **Purpose:** Load all shape data for a collection from disk, parse definitions, and allocate shading tables.
- **Inputs:** Collection index; strip flag (if true, discard bitmap data, keep only metadata)
- **Outputs/Return:** true if successfully loaded and shading tables allocated
- **Side effects:** Reads from `ShapesFile` or `M1ShapesFile`; populates `collection_header->collection`; allocates shading table memory
- **Calls:** `load_collection_definition()`, `load_clut()`, `load_high_level_shape()`, `load_low_level_shape()`, `load_bitmap()`, `allocate_shading_tables()`
- **Notes:** Handles both M1 (resource fork) and M2 (offset-based) formats; color tables and offset tables are parsed from file

### update_color_environment
- **Signature:** `static void update_color_environment(bool is_opengl)`
- **Purpose:** Rebuild all shading tables for all loaded collections by aggregating colors and computing darkening transitions.
- **Inputs:** `is_opengl` flag to determine color format (OpenGL xRGB vs. SDL-mapped RGB)
- **Outputs/Return:** None (modifies global shading table state)
- **Side effects:** Calls `build_shading_tables8/16/32()` for each collection/CLUT, modifies screen color tables, builds tinting tables
- **Calls:** `build_shading_tables8/16/32()`, `build_collection_tinting_table()`, `_change_clut()`, `find_or_add_color()`, `remap_bitmap()`
- **Notes:** Invoked when screen mode changes; processes all loaded collections in order

### build_shading_tables8/16/32
- **Signature:** `static void build_shading_tables{8,16,32}(rgb_color_value *colors, short color_count, pixel{8,16,32} *shading_tables, ...)`
- **Purpose:** Generate bit-depth-specific shading (darkness) lookup tables by interpolating from full color to black.
- **Inputs:** Color palette; output buffer; optional remapping table for 16/32-bit
- **Outputs/Return:** None (fills shading_tables buffer)
- **Side effects:** Modifies shading tables in place
- **Calls:** `get_next_color_run()`, `objlist_set()`, `SDL_MapRGB()` (for 16/32); handles self-luminescent color flag
- **Notes:** 8-bit uses color runs; 16/32-bit use direct RGB interpolation; self-luminescent colors darken less

### load_bitmap
- **Signature:** `static void load_bitmap(std::vector<uint8>& bitmap, SDL_RWops *p, int version)`
- **Purpose:** Parse bitmap metadata and pixel data (RLE or raw) from file into a byte vector.
- **Inputs:** Output vector; file handle; M1 or M2 version flag
- **Outputs/Return:** None (populates bitmap vector with `bitmap_definition` header followed by row pointers and pixel data)
- **Side effects:** Allocates space in bitmap vector; reads from file
- **Calls:** `convert_m1_rle()` (for M1 format); SDL endian conversion functions
- **Notes:** RLE format stores first/last nonblank pixel indices per row; raw format stores full row data

### get_shape_surface (RLE path detail)
- Handles column-major vs. row-major layout via flags
- Applies X/Y mirroring by adjusting destination address and offset directions
- Shrinking performs 2├ù nearest-neighbor downsampling on decompressed pixels

## Control Flow Notes
**Initialization:** `initialize_shape_handler()` ΓåÆ `open_shapes_file()` reads shapes file header; `load_collections()` iterates collection indices, calling `load_collection()` for each marked collection ΓåÆ `update_color_environment()` builds shading tables for rendering.

**Runtime access:** Rendering code calls `get_shape_surface()` to convert shape descriptors to SDL surfaces; alternatively, shape data accessors (`get_low_level_shape_definition()`, `get_bitmap_definition()`, etc.) provide raw pointers for specialized code (OpenGL texturing).

**Patching:** `load_shapes_patch()` parses a patch file with tags (CLDF, HLSH, LLSH, BMAP, CTAB) and selectively replaces collection definitions, high/low-level shapes, bitmaps, and color tables at runtime without reloading from disk.

**Shutdown:** `unload_collection()` frees a collection; `close_shapes_file()` closes file handles.

## External Dependencies
- **SDL2** ΓÇô `SDL_RWops`, `SDL_ReadBE*()`, `SDL_MapRGB()`, `SDL_Surface`, `SDL_CreateRGBSurfaceFrom()`, pixel format queries
- **Game types** ΓÇô `collection_definition`, `bitmap_definition`, `low_level_shape_definition`, `rgb_color_value` (from collection_definition.h)
- **File I/O** ΓÇô `OpenedFile`, `OpenedResourceFile`, `LoadedResource`, `FileHandler.h`
- **Rendering** ΓÇô `OGL_Render.h`, `OGL_LoadScreen.h` for OpenGL integration
- **Configuration** ΓÇô `InfoTree.h` for XML-based infravision tint parsing
- **Utilities** ΓÇô `Packing.h`, `byte_swapping.h`, `cstypes.h`, `cseries.h`
