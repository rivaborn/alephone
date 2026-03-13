# Source_Files/RenderOther/images.cpp

## File Purpose
Manages image resource loading and conversion in Aleph One. Loads PICT resources (Mac classic picture format) from resource files and WAD containers, decompresses various RLE formats, and converts them to SDL surfaces for rendering. Supports multiple color depths and provides picture manipulation (scaling, tiling, scrolling).

## Core Responsibilities
- Open and manage image files from multiple sources (Images, Scenario, External Resources, Shapes, Sounds)
- Parse and decompress PICT opcode streams with PackBits RLE encoding
- Convert PICT resources to SDL surfaces, handling color depths 1/2/4/8/16/32-bit
- Support WAD file format as alternative to Mac resource forks
- Manage color lookup tables (CLUTs) for indexed-color images
- Provide picture scaling and tiling operations
- Special handling for Marathon 1 main menu (composite from shape collection)
- Draw pictures to screen and support scrolling animation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `image_file_t` | class | Wraps opened image file (resource fork or WAD), provides resource access methods |
| `SDL_Surface` | struct | Pixel buffer for images with color format and pitch info |
| `LoadedResource` | class | Wrapper for resource data with automatic cleanup |
| `screen_rectangle` | struct | Portable screen coordinates (top/left/bottom/right) |
| `color_table` | struct | Color palette for indexed-color mode |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ImagesFile` | `image_file_t` | static | Main game images resource file |
| `ScenarioFile` | `image_file_t` | static | Current scenario map's resource file |
| `ExternalResourcesFile` | `image_file_t` | static | External resource file (fallback for Images) |
| `ShapesImagesFile` | `image_file_t` | static | Shapes file images |
| `SoundsImagesFile` | `image_file_t` | static | Sounds file images |
| `m1_menu_unpressed` | `shared_ptr<SDL_Surface>` | static | Cached Marathon 1 unpressed menu composite |
| `m1_menu_pressed` | `shared_ptr<SDL_Surface>` | static | Cached Marathon 1 pressed menu composite |

## Key Functions / Methods

### image_file_t::open_file
- **Signature:** `bool open_file(FileSpecifier &file)`
- **Purpose:** Open an image file, trying resource fork first, then WAD format as fallback
- **Inputs:** FileSpecifier pointing to file on disk
- **Outputs/Return:** true if opened successfully (either format)
- **Side effects:** Opens rsrc_file or wad_file; reads WAD header if applicable
- **Calls:** `FileSpecifier::Open()`, `open_wad_file_for_reading()`, `read_wad_header()`
- **Notes:** Tries both formats; opens WAD as secondary even if resource fork succeeds

### image_file_t::get_pict
- **Signature:** `bool get_pict(int id, LoadedResource &rsrc)`
- **Purpose:** Load PICT resource by ID from either resource fork or WAD
- **Inputs:** Resource ID; output LoadedResource reference
- **Outputs/Return:** true if resource found and loaded
- **Side effects:** Allocates memory for resource data via LoadedResource
- **Calls:** `get_rsrc()` (twice, trying both PICT/pict type codes)

### picture_to_surface
- **Signature:** `std::unique_ptr<SDL_Surface, decltype(&SDL_FreeSurface)> picture_to_surface(LoadedResource &rsrc)`
- **Purpose:** Convert PICT resource to SDL surface by parsing opcodes
- **Inputs:** Loaded PICT resource
- **Outputs/Return:** Unique pointer to SDL surface (or nullptr if failed)
- **Side effects:** Allocates SDL surface; processes and decompresses image data
- **Calls:** `SDL_RWFromMem()`, `uncompress_picture()`, `IMG_LoadTyped_RW()` (for JPEG in opcode 0x8200)
- **Notes:** Handles many PICT opcodes; skips unknown; only converts first image opcode encountered; supports JPEG via SDL_image

### uncompress_picture
- **Signature:** `static int uncompress_picture(const uint8 *src, int row_bytes, uint8 *dst, int dst_pitch, int depth, int height, int pack_type)`
- **Purpose:** Decompress picture data according to depth and pack type; handles color expansion
- **Inputs:** Source buffer, row stride, dest buffer, bit depth, image height, pack type
- **Outputs/Return:** Bytes consumed from source (or -1 on error)
- **Side effects:** Allocates temporary buffer for <8-bit color expansion; frees it afterward
- **Calls:** `uncompress_rle8()`, `uncompress_rle16()`, `uncompress_rle32()`, `byte_swap_memory()`, `malloc()`, `free()`
- **Notes:** Pack type 1=uncompressed, 3=RLE16, 4=RLE32; expands 1/2/4-bit to 8-bit

### unpack_bits (template)
- **Signature:** `template <class T> static const uint8 *unpack_bits(const uint8 *src, int row_bytes, T *dst)`
- **Purpose:** Decompress one scan line via PackBits RLE algorithm; handles byte- or word-sized elements
- **Inputs:** Compressed source, row byte count, destination pointer
- **Outputs/Return:** Updated source pointer after decompression
- **Side effects:** Writes decompressed pixels to destination
- **Notes:** Reads count as 1 or 2 bytes depending on row_bytes; negative flag byte = RLE run, positive = literal run

### get_picture_resource_from_images
- **Signature:** `bool get_picture_resource_from_images(int base_resource, LoadedResource &PictRsrc)`
- **Purpose:** Retrieve PICT resource from Images file, trying bit-depth variants and fallback files
- **Inputs:** Base resource ID; output LoadedResource reference
- **Outputs/Return:** true if found in any file
- **Side effects:** Loads resource into LoadedResource
- **Calls:** `ImagesFile.determine_pict_resource_id()`, `get_pict()` on multiple file objects
- **Notes:** Tries Images file with delta IDs for 16/32-bit, then ExternalResourcesFile, then ShapesImagesFile

### draw_full_screen_pict_resource_from_images
- **Signature:** `void draw_full_screen_pict_resource_from_images(int pict_resource_number)`
- **Purpose:** Load and render a full-screen PICT to the game window
- **Inputs:** Picture resource ID
- **Outputs/Return:** void
- **Side effects:** Renders to screen; special handling for M1 menu reconstruction
- **Calls:** `m1_draw_full_screen_pict_resource_from_images()`, `get_picture_resource_from_images()`, `draw_picture()`

### scroll_full_screen_pict_resource_from_scenario
- **Signature:** `void scroll_full_screen_pict_resource_from_scenario(int pict_resource_number, bool text_block)`
- **Purpose:** Animate a picture scrolling across screen in one or both directions
- **Inputs:** Picture resource ID; text_block flag (affects scroll speed)
- **Outputs/Return:** void
- **Side effects:** Renders to screen repeatedly; loops until done or user input
- **Calls:** `get_picture_resource_from_scenario()`, `picture_to_surface()`, `machine_tick_count()`, `global_idle_proc()`, `yield()`
- **Notes:** Terminates on mouse/key/gamepad input; uses SCROLLING_SPEED constant

### initialize_images_manager
- **Signature:** `void initialize_images_manager(void)`
- **Purpose:** Initialize image system at startup, open Images file, register shutdown handler
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** Opens ImagesFile; calls `atexit()` to register shutdown
- **Calls:** `FileSpecifier::SetNameWithPath()`, `ImagesFile.open_file()`, `atexit()`

### calculate_picture_clut
- **Signature:** `struct color_table *calculate_picture_clut(int CLUTSource, int pict_resource_number)`
- **Purpose:** Get or build color table for picture rendering
- **Inputs:** CLUT source (Images or Scenario); picture resource ID
- **Outputs/Return:** Allocated color_table (must be freed by caller)
- **Side effects:** Allocates color table
- **Calls:** `build_8bit_system_color_table()`, `build_direct_color_table()`, or `get_clut()` + `build_color_table()`
- **Notes:** Currently always returns system color table (TRUE_COLOR_ONLY mode); legacy code for indexed CLUT is disabled

## Control Flow Notes
- **Init**: `initialize_images_manager()` opens main Images file, registers shutdown handler
- **Per-file setup**: Scenario/Shapes/Sounds files set via `set_*_images_file()` functions during level/resource loading
- **Picture loading**: Demand-driven; called from game state handlers and UI code
- **Picture rendering**: Loaded ΓåÆ converted to SDL surface ΓåÆ drawn to screen via `draw_picture_surface()`
- **Shutdown**: `shutdown_images_handler()` closes all open files at application exit

## External Dependencies
- **SDL2/SDL_image**: Graphics surfaces, image codec (JPEG)
- **FileHandler.h**: `FileSpecifier`, `OpenedFile`, `OpenedResourceFile`, `LoadedResource`
- **wad.h**: WAD file format structures and access functions
- **screen_drawing.h**: Drawing primitives, port/surface management
- **interface.h**: Game state, resource IDs (filenameIMAGES, etc.)
- **Plugins.h**: Plugin system for resource override
- **Logging.h**: Debug logging
- **Other includes**: render.h, OGL_Render.h, shell.h (game loop integration)
