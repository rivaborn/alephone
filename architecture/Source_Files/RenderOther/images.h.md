# Source_Files/RenderOther/images.h

## File Purpose
Header file for the image and resource management subsystem in the Aleph One game engine. Provides interfaces for loading, converting, and rendering MacOS PICT resources and managing multiple resource file sources (scenario, shapes, sounds, external). Abstracts platform-specific resource handling and provides SDL surface conversion.

## Core Responsibilities
- Initialize and manage the images/resources manager system
- Query existence of image resources in various file sources
- Manage multiple resource file sources (scenario, shapes, sounds, external images)
- Load picture, sound, and text resources from different file sources
- Compute and build color lookup tables (CLUTs) for palette management
- Render full-screen images and handle scrolling with text overlays
- Convert MacOS PICT resources to SDL surfaces
- Rescale and tile surfaces for various display requirements

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `color_table` | struct (defined elsewhere) | Palette/color lookup table for image rendering |
| `LoadedResource` | class (from FileHandler.h) | RAII wrapper for resource data with automatic cleanup |
| `FileSpecifier` | class (from FileHandler.h) | Abstract file path specification, encapsulates platform-specific paths |
| `SDL_Surface` | struct (SDL2) | Graphics surface representation |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `CLUTSource_Images`, `CLUTSource_Scenario` | enum constants | global | Selector for CLUT source file (images.rsrc or scenario file) |
| Current resource files (scenario, shapes, sounds, external) | managed implicitly | singleton | Multiple open resource files tracked by functions like `set_scenario_images_file`, `set_shapes_images_file` |

## Key Functions / Methods

### initialize_images_manager
- **Signature:** `extern void initialize_images_manager(void);`
- **Purpose:** Initialize the image/resource manager subsystem.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Sets up initial state for resource file management.
- **Calls:** (Implementation not visible.)
- **Notes:** Should be called once at engine startup.

### images_picture_exists / scenario_picture_exists
- **Signature:** `extern bool images_picture_exists(int base_resource);` / `extern bool scenario_picture_exists(int base_resource);`
- **Purpose:** Check if a picture resource exists in the images file or scenario file.
- **Inputs:** `base_resource` ΓÇô resource ID.
- **Outputs/Return:** `bool` ΓÇô true if resource exists.
- **Side effects:** None (read-only query).
- **Calls:** (Implementation not visible.)

### calculate_picture_clut
- **Signature:** `extern struct color_table *calculate_picture_clut(int CLUTSource, int pict_resource_number);`
- **Purpose:** Compute or extract a color lookup table from a PICT resource.
- **Inputs:** `CLUTSource` ΓÇô enum selector (Images or Scenario); `pict_resource_number` ΓÇô resource ID.
- **Outputs/Return:** Pointer to `color_table` struct.
- **Side effects:** Allocates color table (caller responsible for freeing).
- **Calls:** (Implementation not visible.)
- **Notes:** Supports two CLUT sources; allows different palette sources per image.

### set_scenario_images_file / unset_scenario_images_file / set_shapes_images_file / set_external_resources_images_file / set_sounds_images_file
- **Signature:** `extern void set_scenario_images_file(FileSpecifier& File);` (and variants)
- **Purpose:** Register or unregister a resource file as the source for scenario/shapes/external/sounds resources.
- **Inputs:** `File` ΓÇô file specifier.
- **Outputs/Return:** None.
- **Side effects:** Updates global resource file state; subsequent `get_*_resource_from_*` calls will use this file.
- **Calls:** (Implementation not visible.)
- **Notes:** Multiple resource files can be active simultaneously; calling `set_*` replaces the previous file for that category.

### draw_full_screen_pict_resource_from_images / draw_full_screen_pict_resource_from_scenario
- **Signature:** `extern void draw_full_screen_pict_resource_from_images(int pict_resource_number);` / `extern void draw_full_screen_pict_resource_from_scenario(int pict_resource_number);`
- **Purpose:** Load a PICT resource and render it full-screen to the display.
- **Inputs:** `pict_resource_number` ΓÇô resource ID.
- **Outputs/Return:** None.
- **Side effects:** Renders to display; modifies screen state.
- **Calls:** (Implementation not visible, but likely calls `get_picture_resource_from_*` and graphics subsystem.)

### scroll_full_screen_pict_resource_from_scenario
- **Signature:** `extern void scroll_full_screen_pict_resource_from_scenario(int pict_resource_number, bool text_block);`
- **Purpose:** Render a full-screen image with optional text overlay, supporting scrolling.
- **Inputs:** `pict_resource_number` ΓÇô resource ID; `text_block` ΓÇô whether to include a text overlay.
- **Outputs/Return:** None.
- **Side effects:** Renders to display; may display text.
- **Calls:** (Implementation not visible.)
- **Notes:** Used for story/intro screens with scrolling text; text support analogous to M2-Win95 format.

### get_picture_resource_from_images / get_picture_resource_from_scenario / get_sound_resource_from_images / get_sound_resource_from_scenario / get_text_resource_from_scenario
- **Signature:** `extern bool get_picture_resource_from_images(int base_resource, LoadedResource& PictRsrc);` (and variants)
- **Purpose:** Load a resource (picture, sound, or text) from a specified file source into a `LoadedResource` wrapper.
- **Inputs:** `base_resource`/`resource_number` ΓÇô resource ID; `PictRsrc`/`SoundRsrc`/`TextRsrc` ΓÇô output wrapper.
- **Outputs/Return:** `bool` ΓÇô true if load succeeded.
- **Side effects:** Allocates memory for resource data (owned by `LoadedResource`).
- **Calls:** (Implementation not visible, but uses `OpenedResourceFile::Get()`.)
- **Notes:** RAII wrapper ensures automatic cleanup; supports MacOS resource fork emulation.

### picture_to_surface
- **Signature:** `extern std::unique_ptr<SDL_Surface, decltype(&SDL_FreeSurface)> picture_to_surface(LoadedResource &rsrc);`
- **Purpose:** Convert a loaded PICT resource to an SDL_Surface.
- **Inputs:** `rsrc` ΓÇô loaded PICT resource data.
- **Outputs/Return:** `unique_ptr<SDL_Surface>` with SDL_FreeSurface deleter (RAII).
- **Side effects:** Allocates SDL_Surface; decodes PICT format.
- **Calls:** (Implementation not visible, likely SDL and PICT decoder.)
- **Notes:** Returns owning pointer; caller cannot free manually.

### rescale_surface / tile_surface
- **Signature:** `extern SDL_Surface *rescale_surface(SDL_Surface *s, int width, int height);` / `extern SDL_Surface *tile_surface(SDL_Surface *s, int width, int height);`
- **Purpose:** Resize (rescale/stretch or tile/repeat) a surface to specified dimensions.
- **Inputs:** `s` ΓÇô source surface; `width`, `height` ΓÇô target dimensions.
- **Outputs/Return:** New `SDL_Surface*` (caller must free).
- **Side effects:** Allocates new surface.
- **Calls:** (Implementation not visible, uses SDL.)

### find_title_screen / find_m1_title_screen
- **Signature:** `std::unique_ptr<SDL_Surface, decltype(&SDL_FreeSurface)> find_title_screen(FileSpecifier& file);` / `std::unique_ptr<SDL_Surface, decltype(&SDL_FreeSurface)> find_m1_title_screen(FileSpecifier& file);`
- **Purpose:** Locate and load a title screen image from a file (Aleph One format or Marathon 1 format).
- **Inputs:** `file` ΓÇô file specifier to search.
- **Outputs/Return:** `unique_ptr<SDL_Surface>` with SDL_FreeSurface deleter.
- **Side effects:** File I/O; allocates surface.
- **Calls:** (Implementation not visible, likely resource loading + PICT conversion.)
- **Notes:** Searches for title screen by format version; returns owning pointer.

## Control Flow Notes
This is a resource-loading module with no explicit frame loop. Expected usage:
- **Initialization:** Engine calls `initialize_images_manager()` at startup.
- **Setup:** Game code calls `set_scenario_images_file()`, `set_shapes_images_file()`, etc. to register resource sources.
- **Runtime queries:** Rendering/UI code checks resource existence (`images_picture_exists()`) and loads resources (`get_picture_resource_from_scenario()`).
- **Rendering:** Display code calls `draw_full_screen_pict_resource_*()` or converts resources via `picture_to_surface()` for custom rendering.
- **Resource lifecycle:** `LoadedResource` wrappers and `unique_ptr` ensure RAII cleanup; no explicit shutdown required.

## External Dependencies
- **FileHandler.h:** `FileSpecifier`, `LoadedResource`, `OpenedResourceFile` classes.
- **SDL2/SDL.h:** `SDL_Surface`, `SDL_FreeSurface` function.
- **memory:** `std::unique_ptr` template.
- **Standard C library:** Basic types.
- **Implicitly defined elsewhere:** `color_table` struct, resource file implementations, PICT decoder, SDL integration.
