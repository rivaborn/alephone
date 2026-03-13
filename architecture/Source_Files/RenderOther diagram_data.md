# Source_Files/RenderOther/ChaseCam.cpp
## File Purpose
Implements a "chase camera" (third-person camera) system for the Marathon game engine, similar to Halo's behind-view. The camera follows the player from behind, applies damping/spring physics for smooth motion, and performs collision detection to prevent clipping through walls.

## Core Responsibilities
- Manage chase camera activation state and lifecycle (init, reset, activate/deactivate)
- Compute camera position each frame with physics-based damping and springiness
- Perform ray-casting to detect and resolve wall collisions
- Store and retrieve camera orientation (yaw, pitch) synchronized with player
- Handle horizontal camera offset switching (left/right sides)
- Apply inertia effects by tracking previous camera positions across frames

## External Dependencies
- **map.h:** polygon_data, line_data, endpoint_data structures; functions: get_polygon_data(), find_floor_or_ceiling_intersection(), find_line_crossed_leaving_polygon(), get_line_data(), get_endpoint_data(), find_line_intersection(), find_adjacent_polygon()
- **player.h:** player_data structure; extern current_player pointer; world_point3d and angle types
- **network.h:** NetAllowBehindview() (checks network/cheat flags)
- **world.h:** world_point3d, world_point2d, angle type definitions; WORLD_ONE, QUARTER_CIRCLE, SHRT_MAX, SHRT_MIN constants
- **ChaseCam.h:** ChaseCamData structure, GetChaseCamData() function (defined elsewhere, likely preferences.c)

# Source_Files/RenderOther/ChaseCam.h
## File Purpose
This header defines the public interface for a third-person chase camera system (similar to Halo). It enables a follow-behind-the-player camera with configurable physics, optional side-switching for wall avoidance, and optional through-wall rendering. The implementation is distributed across multiple compilation units.

## Core Responsibilities
- Define chase camera state and configuration parameters (`ChaseCamData` struct)
- Provide game initialization and level transition hooks (`ChaseCam_Initialize`, `ChaseCam_Reset`)
- Manage per-frame physics updates (`ChaseCam_Update`)
- Control camera active state and side-switching behavior
- Query camera feasibility and retrieve final view position/orientation for rendering
- Expose configuration dialog and preference loading interfaces

## External Dependencies
- **Includes:** `world.h` (defines `world_point3d`, `angle`, `world_distance`)
- **Implemented elsewhere:**
  - `Configure_ChaseCam()` ΓåÆ PlayerDialogs.c
  - `GetChaseCamData()` ΓåÆ preferences.c
- **Inferred callers:** Game loop, player sprite loader, renderer

# Source_Files/RenderOther/computer_interface.cpp
## File Purpose
Implements the in-game computer terminal interface system where players read messages, view maps, and trigger events. Manages terminal content compilation, rendering, text layout, user input, and state persistence across save/load cycles.

## Core Responsibilities
- Load, parse, and compile Marathon-format terminal text with embedded formatting codes
- Render terminal UI with text, images (PICT resources), checkpoint maps, and visual effects
- Handle player input (keyboard/joystick) for terminal navigation (scroll, page, abort)
- Manage per-player terminal state (current group, line, phase, completion status)
- Pack/unpack terminal data to/from save files and network streams
- Calculate text layout for multiple fonts with word-wrap and line breaking
- Dispatch to specialized rendering for different terminal content types (logon, information, checkpoint, teleport, etc.)

## External Dependencies
- **Includes:** `world.h` (points, polygons), `map.h` (saves_objects), `player.h` (player data), `screen_drawing.h` (rectangle/color/font system), `overhead_map.h` (map rendering), `lua_script.h` (scripting hooks), `sdl_fonts.h`, `joystick.h`, `Packing.h` (stream I/O)
- **External symbols used:** `dynamic_world`, `current_player_index`, `play_object_sound()`, `_render_overhead_map()`, `set_drawing_clip_rectangle()`, `_draw_screen_text()`, `get_interface_rectangle()`, `_get_interface_color()`, `draw_surface` (SDL_Surface), `world_point_to_polygon_index()`, `set_tagged_light_statuses()`, `try_and_change_tagged_platform_states()`, `L_Call_Terminal_Exit()` (Lua), `film_profile` (compatibility config)

# Source_Files/RenderOther/computer_interface.h
## File Purpose
Defines the interface for rendering and managing on-screen computer/terminal interfaces that players interact with in-game. Handles terminal mode initialization, state management, user input, and serialization of terminal data (both map-resident and per-player state).

## Core Responsibilities
- Initialize and manage terminal rendering state per player
- Render terminal UI and text content with formatting (bold, italic, underline)
- Process player input during terminal interaction
- Pack/unpack terminal data for map loading and player state persistence
- Support terminal content sequences (logon, information, checkpoints, sounds, movies, tracks, teleports)
- Query and manage terminal mode activation/deactivation

## External Dependencies
- **cstypes.h**: Fixed-size integer types (`int16`, `uint32`, `uint8`, etc.)

# Source_Files/RenderOther/fades.cpp
## File Purpose

Manages screen fade effects and color table manipulation for the Marathon/Aleph One game engine. Implements time-based visual transitions with various blend modes (tint, dodge, burn, negate, etc.), supports environmental tinting (water/lava/goo), handles gamma correction, and provides both software and OpenGL rendering backends.

## Core Responsibilities

- Initialize, update, and manage fade state with time-based interpolation
- Apply six distinct color manipulation effects with configurable opacity and duration
- Implement priority-based fade scheduling (higher priority fades interrupt lower ones)
- Support independent environmental effects (water, lava, sewage, Jjaro, alien goo) overlaid with fades
- Perform gamma correction on color tables for display calibration
- Integrate OpenGL faders via queue entries for hardware-accelerated effects
- Parse and reset MML-customizable fade definitions and environmental effects

## External Dependencies

- **cseries.h** ΓÇô core macros (PIN, MAX, GetMemberWithBounds), constants (FIXED_ONE, FIXED_FRACTIONAL_BITS).
- **fades.h** ΓÇô public API and enum definitions (fade types, effect types, fader function types).
- **screen.h** ΓÇô color table management (`animate_screen_clut`, `draw_intro_screen`, `world_color_table`, `visible_color_table`, `interface_color_table`).
- **interface.h** ΓÇô game state queries (`get_game_state`).
- **map.h** ΓÇô timing constant (`TICKS_PER_SECOND`, `MACHINE_TICKS_PER_SECOND`); dynamic world state (`dynamic_world->tick_count`).
- **InfoTree.h** ΓÇô XML parser for MML support.
- **OGL_Faders.h** ΓÇô OpenGL fader integration (`OGL_FaderActive`, `GetOGL_FaderQueueEntry`, fader queue indices).
- **Music.h** ΓÇô music system for `full_fade()` idle loop.
- **Movie.h** ΓÇô movie recording integration (`Movie::instance()->AddFrame`).
- **Standard library** ΓÇô `<string.h>`, `<stdlib.h>`, `<math.h>` (pow, memcpy); `<limits.h>` (SHRT_MAX).

# Source_Files/RenderOther/fades.h
## File Purpose
Header file defining the interface for screen fade and color tinting effects in the Marathon/Aleph One game engine. Manages fade-in/fade-out transitions, damage tints (red for bullets, yellow for explosions, etc.), environmental tints (underwater, lava, sewage), and gamma correction. Supports XML-based configuration via MML parser.

## Core Responsibilities
- Define fade types for cinematics, damage feedback, and environmental effects
- Provide API to start, update, and stop fades
- Implement gamma correction for color tables
- Manage fade effect application with configurable delay (MacOS workaround)
- Parse and reset XML-based fade configuration
- Check fade completion and screen state

## External Dependencies
- **`struct color_table`** ΓÇö defined elsewhere; represents indexed color palette
- **`class InfoTree`** ΓÇö C++ class (likely XML parse tree); defined elsewhere
- **MML/XML infrastructure** ΓÇö supports fade configuration declaratively rather than hard-coded

# Source_Files/RenderOther/FontHandler.cpp
## File Purpose
Implements font specification and rendering for both SDL and OpenGL backends. Manages font metrics, precalculates glyph widths, generates OpenGL font textures with display lists, and provides text rendering with alignment and wrapping support.

## Core Responsibilities
- Font specification management (size, style, file parsing with special ID handling)
- Font metric extraction (ascent, descent, leading, line spacing)
- Per-character width precalculation for all 256 character codes
- OpenGL texture atlas generation from SDL surface rendering
- OpenGL display list creation for efficient glyph rendering
- Text rendering with horizontal/vertical alignment and word wrapping
- Registry management for font resources across OpenGL context switches
- Dual-mode rendering (SDL vs. OpenGL) abstraction

## External Dependencies
- **SDL**: `SDL_Surface`, `SDL_CreateRGBSurface()`, `SDL_FillRect()`, `SDL_MapRGB()`, `SDL_FreeSurface()`, `SDL_GetRGB()`
- **OpenGL**: `glGenTextures()`, `glBindTexture()`, `glTexEnvi()`, `glTexParameteri()`, `glTexImage2D()`, `glGenLists()`, `glNewList()`, `glTranslatef()`, `glCallList()`, `glColor4ub()`, `glMatrixMode()`, `glPushMatrix()`, `glPopMatrix()`, `glEnable()`, `glDisable()`, etc.
- **Engine**: `load_font()`, `unload_font()`, `draw_text()` (SDL), `char_width()` (from sdl_fonts / screen_drawing.cpp)
- **Engine**: `OGL_RenderTexturedRect()`, `OGL_Register()`, `OGL_Deregister()`, `MainScreenSurface()`, `MainScreenIsOpenGL()`, `MainScreenSwap()`
- **Engine**: `NextPowerOfTwo()`, `MAX()`, `MIN()` (macros/utilities)
- **Standard**: `<math.h>`, `<string.h>`, `std::set`, `std::string`

# Source_Files/RenderOther/FontHandler.h
## File Purpose
Defines the `FontSpecifier` class for font specification and rendering in Aleph One (a Marathon-like game engine). Provides parameterized font configuration (size, style, file), metric computation, and unified text rendering for both SDL and OpenGL backends with OpenGL texture/display-list management.

## Core Responsibilities
- Store and manage font parameters (size, style, filename, name set)
- Compute and cache derived font metrics (height, ascent, descent, line spacing, character widths)
- Initialize and update fonts from parameters (called by XML parser)
- Render text via SDL surfaces or OpenGL with position/alignment control
- Manage OpenGL font textures, display lists, and filtering parameters
- Track active OpenGL fonts in a static registry for context-switch cleanup
- Provide character and text width queries for layout calculations

## External Dependencies
- **cseries.h:** Core types (`uint8`, `uint16`, `uint32`, `int16`, `short`), macros, basic utilities
- **sdl_fonts.h:** Abstract `font_info` class, SDL/TTF font loading (`load_font()`, `unload_font()`), `TextSpec` struct, font styles (`styleNormal`, `styleBold`, `styleItalic`, `styleUnderline`)
- **OGL_Headers.h:** OpenGL headers (conditional on `HAVE_OPENGL`); provides `GLuint`, `GL_LINEAR`, etc.
- **SDL2/SDL.h:** Via cseries.h; `SDL_Surface` type
- **Standard library:** `std::string`, `std::set`
- **screen_drawing.h:** Referenced for `_draw_screen_text()` behavior model (not included directly)

# Source_Files/RenderOther/game_window.cpp
## File Purpose
Manages the game's heads-up display (HUD) and interface state for the Aleph One engine. Handles HUD buffering, weapon/inventory display configuration, dynamic interface updates, and runtime customization through MML (Marathon Markup Language) parsing.

## Core Responsibilities
- Initialize and update the HUD each frame (software-rendered path)
- Maintain interface state: motion sensor, weapon/ammo, shields, oxygen, inventory
- Define and manage 10 weapon interface layouts with ammunition display parameters
- Buffer HUD to an SDL surface for efficient rendering
- Handle inventory screen scrolling and switching
- Parse and apply MML-based interface customizations at runtime
- Mark interface elements dirty to trigger selective redraws
- Manage motion sensor initialization and state

## External Dependencies
- **SDL2:** `SDL_Surface`, `SDL_CreateRGBSurface`, `SDL_BlitSurface`, `SDL_FillRect`, `SDL_Rect`, `SDL_FreeSurface`
- **HUDRenderer_SW.h:** `HUD_SW_Class`, software HUD rendering
- **screen.h:** `alephone::Screen::instance()`, `RequestDrawingHUD()`, `validate_world_window()`, `game_window_is_full_screen()` (defined elsewhere)
- **shell.h:** `draw_panels()` (extern), `vidmasterStringSetID`, `vidmasterLevelOffset` (defined in shell.cpp)
- **preferences.h:** `GET_GAME_OPTIONS()` macro
- **InfoTree.h:** `InfoTree` class for XML parsing
- **FontHandler.h, screen_definitions.h, images.h, interface_menus.h:** Color, font, and shape definitions
- **Undeclared externs:** `_set_port_to_HUD()`, `_restore_port()`, `initialize_motion_sensor()`, `reset_motion_sensor()`, `current_player_index`, `current_player`, `dynamic_world`, `get_player_data()`, `get_item_kind()`, `calculate_player_item_array()`, `get_interface_rectangle()`, `get_interface_color()`, `get_interface_font()`, `alert_out_of_memory()`, `LuaTexturePaletteSize()`, `get_picture_resource_from_images()`, `picture_to_surface()` (all defined elsewhere in engine).

# Source_Files/RenderOther/game_window.h
## File Purpose
Header declaring the game window initialization, HUD rendering, and interface state management functions. Provides the main entry points for updating and drawing the on-screen HUD each frame, along with dirty-flag functions to invalidate UI elements.

## Core Responsibilities
- Initialize the game window at startup
- Render the HUD and interface each frame (draw and update passes)
- Manage UI element states (ammo, shields, oxygen, weapons, inventory)
- Handle inventory scrolling and manipulation
- Invalidate UI elements via dirty flags to trigger efficient redrawing
- Parse and apply XML-based interface definitions (MML format)
- Ensure HUD framebuffer is allocated and valid

## External Dependencies
- **`Rect`** ΓÇö forward declared; defined elsewhere (likely in graphics/geometry module)
- **`InfoTree`** ΓÇö forward declared; defined elsewhere (likely in XML parsing module)
- Indirectly uses OpenGL (see `OGL_DrawHUD` naming)
- Copyright indicates Bungie Studios / Aleph One engine

# Source_Files/RenderOther/HUDRenderer.cpp
## File Purpose
Implements the HUD rendering system for the Aleph One game engine (Marathon port). Renders player-facing interface elements including shields, oxygen, weapons, ammunition counts, inventory, player names, and network messages. Supports both standard HUD display and debug Lua texture palette visualization.

## Core Responsibilities
- Update and render all HUD elements each frame via dirty-flag optimization
- Render multi-level shield/energy bars with corresponding background textures
- Render oxygen bar with frame-throttled updates (2-tick delay)
- Display current weapon panel with multi-weapon variants and weapon names
- Render ammunition displays (both energy meter and discrete bullet grids)
- Maintain and render inventory panels with item names, quantities, and validity indicators
- Calculate and manage interface rectangle geometry and text layout
- Support Lua script texture palette debug visualization (grid layout of textures)
- Delegate platform-specific drawing operations to virtual methods (DrawShape, DrawText, FillRect, etc.)

## External Dependencies
- **HUDRenderer.h** ΓÇö Base class `HUD_Class`, constants, structure definitions, virtual method interface
- **lua_script.h** ΓÇö Lua texture palette functions: `LuaTexturePaletteSize()`, `LuaTexturePaletteTexture()`, etc.
- **Defined elsewhere** ΓÇö `current_player`, `current_player_index`, `dynamic_world`, `interface_state`, `weapon_interface_definitions`, shape/texture IDs (_energy_bar, _weapon_panel_shape, etc.), UI helper functions (`get_interface_rectangle()`, `get_player_desired_weapon()`, `calculate_player_item_array()`, etc.), drawing primitives (`DrawShape`, `DrawText`, `FillRect`, `FrameRect` ΓÇö all virtual, subclass-implemented)

# Source_Files/RenderOther/HUDRenderer.h
## File Purpose
Base class and data structures for rendering the heads-up display (HUD) in the Aleph One game engine. Defines interface for weapon panels, ammo displays, health/oxygen bars, motion sensors, and inventory visualization. Serves as the architectural foundation for platform-specific HUD implementations.

## Core Responsibilities
- Define abstract HUD rendering interface via `HUD_Class` base class
- Manage weapon panel display configuration and data structures
- Track HUD element dirty states to optimize redraw performance
- Coordinate updates for suit energy, oxygen, weapons, ammunition, and inventory
- Provide virtual methods for platform-specific shape drawing, text rendering, and clipping
- Define shape descriptor enums for HUD textures (bars, weapon icons, motion sensor elements)

## External Dependencies
- **cseries.h**: Core platform abstraction (fixed-point types, basic macros, SDL headers)
- **map.h**: `TICKS_PER_SECOND`, shape descriptors, world geometry
- **interface.h**: Shape/animation data structures, collection management
- **player.h**: Player data structures, weapon and item types
- **SoundManager.h**: Sound playback for button clicks and alerts
- **motion_sensor.h**: Motion sensor shape descriptors and initialization
- **items.h**: Item type enums (powerups, ammo, weapons)
- **weapons.h**: Weapon definitions and ammo display info
- **network_games.h**: Network compass shape descriptors
- **screen_drawing.h**: Screen coordinate and clipping utilities

**Defined Elsewhere**: `shape_descriptor`, `screen_rectangle`, `point2d` (imported from interface/screen headers); player/weapon/item state queried from global `current_player` and map data.

# Source_Files/RenderOther/HUDRenderer_Lua.cpp
## File Purpose
Implements the Lua HUD renderer, providing C++ bindings and dual-path (OpenGL/SDL) rendering for Lua-scripted HUD themes. Manages motion sensor blips, drawing primitives (rectangles, text, images, shapes), and stencil-based masking modes for HUD composition.

## Core Responsibilities
- **Blip management**: track entity radar markers with position, intensity, and type
- **Rendering state**: initialize/finalize OpenGL or SDL surfaces for HUD frame
- **Masking modes**: manage stencil test state for compositing (disabled/enabled/drawing/erasing)
- **Clipping**: apply scissor test (OpenGL) or clip rects (SDL) to drawing operations
- **Drawing primitives**: filled/framed rectangles, text, pre-rendered images, game shapes
- **Dual rendering paths**: transparent abstraction over OpenGL (shader-less, immediate mode) and SDL software rendering
- **Coordinate transform**: translate HUD coordinates to screen space; matrix management for OpenGL

## External Dependencies
- **FontHandler.h**: `FontSpecifier` class for text rendering
- **Image_Blitter.h**, **Shape_Blitter.h**: Sprite/bitmap rendering abstractions
- **lua_hud_script.h**: Lua callback entry point `L_Call_HUDDraw()`
- **shell.h**: `get_screen_mode()`, `MainScreenSurface()`, etc.
- **screen.h**: `alephone::Screen` singleton, viewport/clip management
- **images.h**: Resource loading (used elsewhere, not directly here)
- **OGL_Headers.h / OGL_Render.h**: OpenGL state, `OGL_RenderRect()`, `OGL_RenderFrame()`
- **SDL2/SDL.h**: Surface creation, blitting, pixel format conversion
- **math.h**: `ceilf()` (disabled scaling code)
- External symbols: `MotionSensorActive` (bool, state flag); `distance2d()`, `arctangent()` (geometry, defined elsewhere); `L_Call_HUDDraw()` (Lua)

# Source_Files/RenderOther/HUDRenderer_Lua.h
## File Purpose
Implements a Lua-compatible HUD renderer class that extends the base `HUD_Class`. Provides drawing primitives and motion sensor blip management for scripted HUD themes, bridging Lua script logic with native rendering operations.

## Core Responsibilities
- Manage entity blips (radar markers) for motion sensor display
- Provide drawing primitives (filled/framed rectangles, text, images, shapes)
- Handle masking modes for clipped rendering regions
- Update and render motion sensor display state per frame
- Maintain drawing context (start/end draw, apply clipping)
- Track drawing/masking state across render lifecycle

## External Dependencies
- **Base class**: `HUD_Class` (HUDRenderer.h) ΓÇö provides `update_everything()` and virtual method framework
- **SDL**: `SDL_Surface`, `SDL_Rect` ΓÇö software rendering surfaces
- **Forward declarations**: `FontSpecifier`, `Image_Blitter`, `Shape_Blitter` ΓÇö game-specific rendering abstraction classes
- **Game types** (defined elsewhere): `world_distance`, `angle`, `shape_descriptor`, `screen_rectangle`, `point2d`
- **Exceptions**: Throws `std::logic_error` from `TextWidth()` override to indicate Lua path does not use it

# Source_Files/RenderOther/HUDRenderer_OGL.cpp
## File Purpose
Implements OpenGL-based rendering for the HUD (Heads-Up Display), including dynamic UI elements, text, shapes, motion sensor, and interface graphics. Provides the primary HUD rendering pipeline integrating texture management, font rendering, and geometric primitives.

## Core Responsibilities
- Main HUD rendering orchestration (backdrop, dynamic elements, matrix setup)
- Motion sensor visualization with circular clipping
- Shape rendering with texture mapping and transparency control
- Text rendering with color and font specifications
- Primitive geometry (filled/framed rectangles)
- Texture loading and lifecycle management for HUD assets
- OpenGL state management (attributes, matrices, blending modes)

## External Dependencies
- **OpenGL API:** `glPushAttrib`, `glPopAttrib`, `glDisable`, `glEnable`, `glMatrixMode`, `glPushMatrix`, `glPopMatrix`, `glLoadIdentity`, `glTranslated`, `glScaled`, `glColor3*`, `glClipPlane`
- **Rendering modules:** `OGL_RenderRect()`, `OGL_RenderFrame()`, `OGL_RenderTexturedRect()` (defined elsewhere)
- **Texture management:** `OGL_Blitter`, `TextureManager`, `Shape_Blitter`, `TxtrTypeInfoList[OGL_Txtr_HUD]`
- **Font system:** `FontHandler.h`, `get_interface_font()`, `FontSpecifier`
- **Game state:** `MotionSensorActive`, `GET_GAME_OPTIONS()`, `current_player_index`, game option flags
- **Interface resources:** `get_interface_color()`, `get_interface_rectangle()`, shape descriptors, interface collections
- **Utility:** `get_shape_bitmap_and_shading_table()`, `LuaTexturePaletteSize()` (Lua integration), `BUILD_DESCRIPTOR()` macros

# Source_Files/RenderOther/HUDRenderer_OGL.h
## File Purpose
OpenGL-specific implementation of HUD rendering for the Aleph One game engine. Provides concrete methods for drawing HUD elements (motion sensor, shapes, text, UI rectangles) using OpenGL graphics calls.

## Core Responsibilities
- Update and render the motion sensor display with entity blips
- Draw shapes and textures at specified screen coordinates
- Render text with font and color support
- Fill and frame rectangles for UI elements
- Manage OpenGL clipping planes (e.g., for circular motion sensor viewport)
- Calculate text width for layout purposes
- Handle transparency and shape visibility in HUD rendering

## External Dependencies
- `HUDRenderer.h` (base class `HUD_Class`, type definitions: `shape_descriptor`, `screen_rectangle`, `point2d`)
- OpenGL API (called in implementation .cpp)
- Types defined in bundled headers: `shape_descriptor`, `screen_rectangle`, `point2d`

# Source_Files/RenderOther/HUDRenderer_SW.cpp
## File Purpose
Software-rasterized HUD rendering implementation for the Aleph One game engine. Provides concrete implementations of the `HUD_Class` interface using SDL surfaces, enabling 2D UI composition via shape blitting, text rendering, and primitive drawing operations.

## Core Responsibilities
- Update and render the motion sensor indicator when game state changes
- Draw 2D shapes from the shapes resource file via delegation to screen drawing functions
- Draw scaled textured elements (e.g., weapon textures, HUD decorations) using `Shape_Blitter`
- Render text strings with color and font selection
- Draw filled and outlined rectangles for UI primitives
- Rotate SDL surfaces 90┬░ for orientation transformations
- Serve as the software rendering backend, complementary to an OpenGL path

## External Dependencies
- **Includes:** `HUDRenderer_SW.h` (class definition), `images.h` (image resource access), `shell.h` (shape surface retrieval), `Shape_Blitter.h` (textured shape rendering)
- **External functions:** `_draw_screen_shape()`, `_draw_screen_shape_at_x_y()`, `_draw_screen_text()`, `_text_width()`, `_fill_rect()`, `_frame_rect()`, `motion_sensor_has_changed()`, `render_motion_sensor()`, `get_interface_rectangle()`, `GET_GAME_OPTIONS()`, `BUILD_DESCRIPTOR()`, `GET_COLLECTION()`, `GET_DESCRIPTOR_COLLECTION()`, `GET_DESCRIPTOR_SHAPE()`, `GET_COLLECTION_CLUT()`
- **SDL symbols:** `SDL_CreateRGBSurface()`, `SDL_SetPaletteColors()`, `SDL_Rect`, `SDL_Surface`, `SDL_FreeSurface()`
- **External state:** `HUD_Buffer` (target drawing surface), `MotionSensorActive` (motion sensor enabled flag)

# Source_Files/RenderOther/HUDRenderer_SW.h
## File Purpose
Software (CPU/memory-based) implementation of HUD rendering for Aleph One. Provides concrete graphics operations using screen drawing primitivesΓÇöan alternative to hardware/GPU-accelerated HUD rendering. Written in 2001 as part of the Bungie/Aleph One codebase.

## Core Responsibilities
- Override pure virtual base methods from `HUD_Class` with software rendering implementations
- Update and render motion sensor (radar) display and entity blips
- Draw shapes, text, and rectangles at screen coordinates
- Manage texture rendering and clipping regions
- Calculate text dimensions for layout purposes
- Handle shape drawing with optional transparency and source/dest clipping

## External Dependencies
- **Includes:** `HUDRenderer.h` (base class, constants, data structures)
- **Defined elsewhere:** `shape_descriptor`, `screen_rectangle`, `point2d` types; shape/texture/font/color constants; `SoundManager`, `motion_sensor`, player/weapon/item subsystems (included transitively via HUDRenderer.h)

# Source_Files/RenderOther/Image_Blitter.cpp
## File Purpose
Implements the `Image_Blitter` class for rendering 2D images in the game UI. Manages SDL surface lifecycle, image loading from multiple sources, rescaling, and drawing with support for tinting and rotation.

## Core Responsibilities
- Load images from `ImageDescriptor` objects, resource IDs, or SDL_Surface objects
- Manage SDL surface memory (creation, conversion, rescaling, cleanup)
- Maintain image state: source dimensions, scaled dimensions, and crop rectangle
- Handle on-demand surface conversion and rescaling during draw operations
- Provide dimension queries (scaled and unscaled)
- Draw images to destination surfaces with optional source cropping

## External Dependencies
- **SDL2**: `SDL_Surface`, `SDL_Rect`, `SDL_CreateRGBSurfaceFrom()`, `SDL_CreateRGBSurface()`, `SDL_FreeSurface()`, `SDL_SetSurfaceBlendMode()`, `SDL_BlitSurface()`, `SDL_ConvertSurfaceFormat()`
- **ImageLoader** (header): `ImageDescriptor`
- **images.h**: `get_picture_resource_from_images()`, `picture_to_surface()`, `rescale_surface()`
- **Platform detection**: `PlatformIsLittleEndian()`
- **cseries.h** (via Image_Blitter.h): Type definitions (`uint32`)

# Source_Files/RenderOther/Image_Blitter.h
## File Purpose
Defines the `Image_Blitter` class for rendering 2D UI images to SDL surfaces with support for scaling, cropping, tinting, and rotation. Part of the Aleph One game engine's 2D rendering system.

## Core Responsibilities
- Load images from `ImageDescriptor`, resource IDs, or SDL surfaces with optional source cropping
- Manage scaled and original image buffers
- Render images to destination surfaces with transformations
- Apply per-image tinting and rotation effects
- Expose image dimensions (scaled and unscaled)

## External Dependencies
- **SDL2:** `SDL_Surface`, `SDL_Rect` (graphics primitives)
- **ImageLoader.h:** `ImageDescriptor` (cross-format image container)
- **cseries.h:** Standard engine utilities and includes
- **STL:** `<vector>`, `<set>` (declared but not used in header)

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

## External Dependencies
- **SDL2/SDL_image**: Graphics surfaces, image codec (JPEG)
- **FileHandler.h**: `FileSpecifier`, `OpenedFile`, `OpenedResourceFile`, `LoadedResource`
- **wad.h**: WAD file format structures and access functions
- **screen_drawing.h**: Drawing primitives, port/surface management
- **interface.h**: Game state, resource IDs (filenameIMAGES, etc.)
- **Plugins.h**: Plugin system for resource override
- **Logging.h**: Debug logging
- **Other includes**: render.h, OGL_Render.h, shell.h (game loop integration)

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

## External Dependencies
- **FileHandler.h:** `FileSpecifier`, `LoadedResource`, `OpenedResourceFile` classes.
- **SDL2/SDL.h:** `SDL_Surface`, `SDL_FreeSurface` function.
- **memory:** `std::unique_ptr` template.
- **Standard C library:** Basic types.
- **Implicitly defined elsewhere:** `color_table` struct, resource file implementations, PICT decoder, SDL integration.

# Source_Files/RenderOther/motion_sensor.cpp
## File Purpose
Implements the motion sensor HUD elementΓÇöa circular radar display showing nearby monsters, players, and network compass indicators. Tracks entity positions, manages visibility/fade animation, and renders colored blips for friendly, alien, and enemy targets. Supports software and OpenGL rendering backends.

## Core Responsibilities
- Entity tracking: maintain array of "blips" for monsters/players within sensor range
- Periodic scanning: rescan game world for entities within `MOTION_SENSOR_RANGE` 
- Position updates: shift entity history each frame and detect visibility changes
- Blip rendering: draw entity sprites with intensity/fade based on position history
- Network compass: display team beacon directions in multiplayer games
- Bitmap operations: sprite clipping and copying for software renderer
- Fade-out logic: gracefully remove entities by animating signal decay
- MML configuration: parse XML settings for frequency, range, scale, and monster display types

## External Dependencies
- **Map/World:** `map.h` (objects, polygons, world coordinates), `monsters.h` (monster_data), `player.h` (player_data)
- **Rendering:** `render.h` (view_data), `interface.h` (shape descriptors, bitmap_definition, shape lookup functions)
- **Network:** `network_games.h` (get_network_compass_state)
- **HUD Renderers:** `HUDRenderer_SW.h`, `HUDRenderer_OGL.h`, `HUDRenderer_Lua.h` (base classes and rendering methods)
- **Config:** `InfoTree.h` (XML parsing for MML)
- **Math/Utility:** `cseries.h` (standard types), `<math.h>`, `<string.h>`, `<stdlib.h>`

**Defined elsewhere:** `get_object_data()`, `get_monster_data()`, `get_player_data()`, `guess_distance2d()`, `transform_point2d()`, `get_shape_bitmap_and_shading_table()`, `get_monster_index_to_player_index()`, `MotionSensorActive` (external bool), `m1_solo_player_in_terminal()`, shape-related accessors.

# Source_Files/RenderOther/motion_sensor.h
## File Purpose
Header for the motion sensor (radar/minimap) system in the Aleph One game engine. Declares types and functions to initialize, scan, and update the motion sensor display, which shows friendly units, aliens, and enemies at different marker types.

## Core Responsibilities
- Define motion sensor display types (friend/alien/enemy classification)
- Declare motion sensor initialization with graphical assets
- Declare scanning and state management functions
- Provide change-detection for display updates
- Support runtime configuration via MML (Marathon Markup Language)
- Manage motion sensor range adjustments

## External Dependencies
- **shape_descriptors.h**: Provides `shape_descriptor` typedef (packed uint16 encoding collection/shape/CLUT indices).
- **InfoTree** (class): Forward-declared; defined elsewhere (likely XML/markup parsing utility).

# Source_Files/RenderOther/OGL_Blitter.cpp
## File Purpose
Implements efficient 2D image rendering via OpenGL by tiling SDL surfaces into power-of-two GPU textures. Manages texture lifecycle and provides lazy-loaded rendering with support for scaling, rotation, and per-instance tinting.

## Core Responsibilities
- Decompose source images into tile-aligned chunks matching GPU constraints
- Load SDL surface pixels into OpenGL 2D textures with configurable filtering
- Render tiled texture rectangles with coordinate transformation and clipping
- Maintain a global registry of active blitters for coordinated cleanup on GL context loss
- Handle edge-pixel smearing to eliminate texture sampling artifacts at tile boundaries
- Support view transformations (scaling, rotation) in the render pipeline

## External Dependencies
- **OGL_Setup.h** ΓÇô Get_OGL_ConfigureData(), PlatformIsLittleEndian(), OpenGL constants
- **OGL_Render.h** ΓÇô OGL_RenderTexturedRect() helper
- **screen.h** ΓÇô alephone::Screen singleton, MainScreenLogicalWidth/Height()
- **shell.h** ΓÇô shell utilities
- **&lt;memory&gt;** ΓÇô std::unique_ptr with SDL_FreeSurface deleter
- **SDL2** ΓÇô SDL_Surface, SDL_Rect, SDL_CreateRGBSurface(), SDL_BlitSurface(), SDL_SetSurfaceBlendMode()
- **OpenGL (global)** ΓÇô glGenTextures(), glBindTexture(), glTexParameteri(), glTexImage2D(), glDeleteTextures(), GL_TEXTURE_2D, blend/matrix ops
- **Parent class Image_Blitter** ΓÇô provides Loaded(), m_surface, m_src, m_scaled_src, crop_rect, rotation, tint_color_{r,g,b,a}

# Source_Files/RenderOther/OGL_Blitter.h
## File Purpose
Declares `OGL_Blitter`, an OpenGL-based image blitter that inherits from `Image_Blitter` to render 2D images to screen. Manages OpenGL texture creation, tiling across multiple 256├ù256 tiles, and texture lifecycle on context switches via a static registry of active blitters.

## Core Responsibilities
- Create, load, and unload OpenGL textures for image blitting
- Provide `Draw()` method overloads to render images with source/destination rectangles
- Tile large images across multiple 256├ù256 textures and track them in `m_refs`
- Maintain a static registry (`m_blitter_registry`) to track all active blitters for safe context switching
- Expose static screen/viewport management: binding, coordinate conversion, dimension queries
- Support configurable texture filtering via `nearFilter` parameter

## External Dependencies
- `cseries.h` ΓÇô platform/utility macros
- `Image_Blitter.h` ΓÇô base class with `crop_rect`, tint, rotation, surface management
- `ImageLoader.h` ΓÇô `ImageDescriptor` type (not directly used in header, but implied for image loading)
- `OGL_Headers.h` ΓÇô OpenGL typedefs (`GLuint`), platform-specific GL setup
- `<vector>`, `<set>` ΓÇô STL containers
- `SDL_Surface`, `SDL_Rect` ΓÇô SDL2 types

# Source_Files/RenderOther/OGL_LoadScreen.cpp
## File Purpose
Implements OpenGL-based load screen rendering for the Aleph One game engine. Displays splash/loading images with optional progress bars during asset loading or level transitions. Manages screen initialization, image display, progress updates, and cleanup via a singleton pattern.

## Core Responsibilities
- Manage singleton instance of the load screen system
- Load and validate image files for display
- Configure image scaling and centering based on aspect ratio and screen dimensions
- Render progress bars with configurable position, size, and colors
- Coordinate OpenGL buffer operations (clear, swap) for proper frame display
- Calculate transformation matrices for image and progress bar positioning
- Handle configuration state (paths, scaling modes, progress settings)

## External Dependencies
- **OGL_Render.h:** `OGL_ClearScreen()`, `OGL_SwapBuffers()`, `OGL_RenderRect()`
- **OGL_Blitter.h:** `OGL_Blitter` class, `OGL_Blitter::BoundScreen()`
- **ImageLoader.h:** `ImageDescriptor`, `ImageLoader_Colors` enum constant
- **screen.h:** `alephone::Screen::instance()`, `Screen::bound_screen()`
- **OpenGL (via included headers):** `glMatrixMode()`, `glPushMatrix()`, `glTranslated()`, `glScaled()`, `glColor3us()`, `glPopMatrix()`
- **SDL2:** `SDL_Rect` type
- **Standard library:** `std::string` for file paths

# Source_Files/RenderOther/OGL_LoadScreen.h
## File Purpose
Defines a singleton class that manages OpenGL-based load screens for the game engine. Handles displaying background images, progress indicators, and visual feedback during asset loading phases.

## Core Responsibilities
- Singleton instance management and lifecycle (Start/Stop)
- Loading and configuring load screen images with flexible positioning and scaling
- Rendering progress indicators with customizable colors
- Managing OpenGL texture resources and blitting parameters
- Querying active state and providing color palette access for rendering

## External Dependencies
- **cseries.h** ΓÇö Core type definitions, SDL includes
- **OGL_Headers.h** ΓÇö OpenGL context and function declarations
- **OGL_Blitter.h** ΓÇö Image blitting to OpenGL framebuffer
- **ImageLoader.h** ΓÇö `ImageDescriptor` class for image data storage
- **SDL2** ΓÇö Rect structures and surface abstractions

# Source_Files/RenderOther/overhead_map.cpp
## File Purpose
Implements the overhead map (automap/motion sensor) rendering system for the Marathon game engine. Manages configuration, font initialization, and delegates rendering to platform-specific implementations (SDL or OpenGL).

## Core Responsibilities
- Initialize and manage overhead map rendering configuration with default colors, line styles, and entity definitions
- Lazy-initialize fonts for map annotations and titles
- Route rendering calls to appropriate backend (software SDL or OpenGL) based on `OGL_MapActive` flag
- Reset automap visibility data based on current mode (Normal, CurrentlyVisible, or All)
- Parse and apply XML configuration for overhead map display parameters (colors, fonts, visibilities)
- Maintain MML (Marathon Markup Language) configuration state for reset/reload cycles

## External Dependencies
- **Notable includes:** `cseries.h` (platform utilities), `shell.h` (`_get_player_color`), `map.h` (map constants, `dynamic_world`), `monsters.h` (monster type enums), `OverheadMap_SDL.h`, `OverheadMap_OGL.h` (renderer implementations), `InfoTree.h` (XML parsing)
- **Defined elsewhere:** `automap_lines`, `automap_polygons`, `dynamic_world`, `NUMBER_OF_MONSTER_TYPES`, `NUMBER_OF_COLLECTIONS`, `OVERHEAD_MAP_MAXIMUM_SCALE`, `OVERHEAD_MAP_MINIMUM_SCALE`, `OverheadMapClass`, `OverheadMap_SDL_Class`, `OverheadMap_OGL_Class`

# Source_Files/RenderOther/overhead_map.h
## File Purpose
Defines the interface for rendering a tactical/overhead map overlay during gameplay and saved game previews. Provides configuration constants, map state management, and rendering entry points with MML-based customization support.

## Core Responsibilities
- Define overhead map scaling limits and defaults (scale 1ΓÇô4, default 3)
- Manage rendering modes: saved game preview, checkpoint map, live game map
- Store overhead map rendering state (scale, origin point, screen position/dimensions)
- Expose rendering function to draw the scaled top-down map view
- Support MML (XML) configuration parsing and reset for customization

## External Dependencies
- `#include "world.h"` ΓÇö Provides `world_point2d`, `world_distance`, angle types, and world coordinate constants (WORLD_ONE, etc.)
- `InfoTree` class ΓÇö Forward-declared; XML/MML parse tree (defined elsewhere, likely in config parsing system)
- Implied: render system (framebuffer, drawing primitives), game world/polygon data

# Source_Files/RenderOther/OverheadMap_OGL.cpp
## File Purpose
OpenGL implementation of overhead map (automap) rendering for Marathon game engine. Provides hardware-accelerated 2D map display with cached geometry batching for performance optimization.

## Core Responsibilities
- Clear screen and manage OpenGL state for 2D overhead map rendering
- Implement polygon rendering with color-change batching (triangle fans)
- Implement line rendering with width and color caching
- Render game world objects (things) as circles or rectangles at specified sizes
- Render player position and orientation as triangles with proper transformation
- Render map annotations (text) with font support and justification
- Render monster/npc paths as line strips
- Optimize rendering via geometry caching (flush on parameter changes)

## External Dependencies
- **OpenGL**: `glColor3usv()`, `glColor4us()`, `glMatrixMode()`, `glPushMatrix()`, `glPopMatrix()`, `glTranslatef()`, `glRotatef()`, `glScalef()`, `glVertexPointer()`, `glDrawElements()`, `glDrawArrays()`, state queries/changes
- **OGL_Render.h**: `OGL_RenderRect()`, `OGL_RenderLines()`, extern `ViewWidth`/`ViewHeight`
- **OGL_Textures.h**: `TxtrTypeInfoList[]` (for HUD texture filter info)
- **OverheadMapRenderer.h**: Base class `OverheadMapClass` (parent implementation not provided)
- **map.h**: `rgb_color`, `world_point2d`, `world_point3d`, `angle` type definitions
- **screen.h**: `map_is_translucent()` function
- **FontSpecifier**: Font rendering interface (defined elsewhere)

# Source_Files/RenderOther/OverheadMap_OGL.h
## File Purpose
OpenGL-specific implementation of the overhead map (mini-map) renderer for the Aleph One game engine. Subclass of `OverheadMapClass` that provides GPU-accelerated rendering of game map elements, entities, and annotations.

## Core Responsibilities
- Implement OpenGL rendering for map polygons, lines, entities, and text
- Cache geometry data (polygons, lines, paths) for efficient batch rendering to GPU
- Render game entities (monsters, items, projectiles, checkpoints) on the overhead map
- Draw the player character with facing direction
- Support text annotation rendering with font specifications
- Implement begin/end batching lifecycle for each geometry type
- Manage path visualization for debugging/development

## External Dependencies
- `<vector>` (STL containers)
- `OverheadMapRenderer.h` (base class `OverheadMapClass`, config data structures, type definitions)
- Engine types: `world_point2d`, `angle`, `rgb_color`, `FontSpecifier` (defined elsewhere)

# Source_Files/RenderOther/OverheadMap_SDL.cpp
## File Purpose
Implements SDL-based rendering for the overhead (minimap) display in Aleph One. Provides concrete drawing methods for map elements: geometry, objects, players, text labels, and path trails, inheriting from `OverheadMapClass`.

## Core Responsibilities
- Convert map vertex indices to world coordinates and rasterize polygons
- Render lines, game objects (rectangles/circles), and player indicators
- Draw text annotations with font and justification control
- Manage incremental path drawing for player trails
- Map RGB color values to SDL pixel format

## External Dependencies
- **SDL2**: `SDL_MapRGB()`, `SDL_FillRect()`, `SDL_Surface`, `SDL_Rect`
- **Aleph One core**: `world_point2d`, `rgb_color`, `angle`, `FontSpecifier`
- **External functions**: `::draw_polygon()`, `::draw_line()`, `::draw_text()` (from screen_drawing.h; rasterization backends)
- **Base class**: `GetVertex()` (OverheadMapClass), `text_width()` (font utility)
- **Global**: `draw_surface` (extern SDL_Surface* from screen_sdl.cpp)

# Source_Files/RenderOther/OverheadMap_SDL.h
## File Purpose
SDL-specific subclass of OverheadMapClass that implements the abstract rendering interface using Simple DirectMedia Layer. Provides concrete implementations for drawing polygons, lines, entities, player indicators, text, and path visualization on the overhead (mini) map.

## Core Responsibilities
- Implement SDL rendering pipeline for overhead map polygons
- Render line segments (walls, elevation changes, control panels)
- Draw game entities (monsters, items, projectiles, checkpoints)
- Render player representation with directional facing indicators
- Render text annotations at map locations with font specification
- Manage path drawing state and incremental path rendering

## External Dependencies
- **Inherits from:** `OverheadMapClass` (OverheadMapRenderer.h)
- **Type dependencies:** `rgb_color`, `world_point2d`, `angle` (from world.h), `FontSpecifier` (from FontHandler.h)
- **Not included in this header:** SDL itself (assumed in .cpp implementation)

# Source_Files/RenderOther/OverheadMapRenderer.cpp
## File Purpose
Implements the base overhead-map (automap) rendering system for Marathon. Transforms world coordinates to screen space and renders game-world features (polygons, lines, entities, annotations) from a top-down view, with special support for limited-visibility checkpoint maps via flood-fill algorithms.

## Core Responsibilities
- Main render loop: coordinates overall automap drawing via `begin_*()` / `end_*()` virtual hooks
- Viewport transforms: converts world coordinates to screen space using fixed-point bit-shift scaling
- Polygon coloring: determines colors based on type (platform, water, lava, etc.), media presence, and secret status
- Line visibility: draws elevation/solid edges between polygons with proper ownership checks
- Entity rendering: draws players (with team colors), monsters, items, projectiles with visibility filtering
- Annotation/text: positions and renders map labels at appropriate scales
- Path visualization: renders AI pathfinding waypoints when enabled
- Checkpoint automap: generates/restores limited visibility maps via flood-fill cost function and state save/restore

## External Dependencies
- **Includes/Headers:**
  - `cseries.h` ΓÇö cross-platform base types, macros
  - `OverheadMapRenderer.h` ΓÇö class definition, color/shape/font configuration data structures
  - `flood_map.h` ΓÇö `flood_map()` algorithm and cost function typedef
  - `media.h` ΓÇö media type enums (`_media_water`, etc.)
  - `platforms.h` ΓÇö platform accessors and flag macros
  - `player.h` ΓÇö player data accessors, team color constants
  - `render.h` ΓÇö rendering flag macros, constants
  - `<string.h>`, `<stdlib.h>`, `<limits.h>` ΓÇö standard C library

- **Game-World Symbols (defined elsewhere):**
  - `dynamic_world` ΓÇö current world state (polygons, lines, endpoints, objects, tick count)
  - `static_world` ΓÇö static level data (level name)
  - `objects`, `saved_objects` ΓÇö entity and saved-object arrays
  - `automap_lines`, `automap_polygons` ΓÇö automap visibility bitfields (global)
  - Accessor functions: `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`, `get_media_data()`, `get_platform_data()`, `get_monster_data()`, `get_player_data()`, `get_next_map_annotation()`, `get_number_of_paths()`, `path_peek()`, `find_flooding_polygon()`, `monster_index_to_player_index()`, `GET_GAME_OPTIONS()`, various flag test/set macros

- **Virtual Methods (implemented by subclasses):**
  - `begin_overall()`, `end_overall()` ΓÇö rendering lifecycle
  - `begin_polygons()`, `end_polygons()`, `draw_polygon()`
  - `begin_lines()`, `end_lines()`, `draw_line()`
  - `begin_things()`, `end_things()`, `draw_thing()`
  - `draw_player()`, `draw_text()`, `draw_annotation()`, `draw_map_name()`
  - `set_path_drawing()`, `draw_path()`, `finish_path()`

# Source_Files/RenderOther/OverheadMapRenderer.h
## File Purpose
Base class and configuration structures for overhead map (minimap/automap) rendering. Provides virtual methods for graphics-API-agnostic rendering and defines styling data (colors, fonts, shapes) for all automap elements including polygons, lines, entities, and annotations.

## Core Responsibilities
- Define configuration structures (`OvhdMap_CfgDataStruct`) for automap appearance (polygon/line colors, entity shapes, fonts, paths)
- Provide virtual base class (`OverheadMapClass`) for graphics-API-specific renderer implementations
- Define rendering pipeline with virtual hooks (begin/end polygons, lines, things, text, paths)
- Handle coordinate transformation for map viewport rendering
- Support false automap generation (checkpoint/saved-game preview modes)
- Expose static accessor methods for vertex data to subclasses (esp. OpenGL)

## External Dependencies
- **Notable includes**: 
  - `cseries.h` ΓÇô Core types (rgb_color, RGBColor, angle)
  - `world.h` ΓÇô World geometry (world_point2d, world_distance, angle)
  - `map.h` ΓÇô Map structures (endpoint_data, line_data via `get_line_data()`, etc.)
  - `monsters.h` ΓÇô Monster types
  - `shape_descriptors.h` ΓÇô Shape descriptor macros
  - `shell.h` ΓÇô Platform I/O (_get_player_color)
  - `FontHandler.h` ΓÇô FontSpecifier for text rendering
- **Defined elsewhere**:
  - `get_endpoint_data()`, `get_line_data()` ΓÇô Accessors to map geometry
  - `overhead_map_data` ΓÇô Viewport specification (defined in `overhead_map.h`)
  - `endpoint_data.transformed` ΓÇô Cached viewport-space vertex coordinates
  - `_render_overhead_map()` ΓÇô Likely calls `Render()` on a concrete subclass instance

# Source_Files/RenderOther/screen.cpp
## File Purpose
Main display and window management system for Aleph One's SDL2-based renderer. Orchestrates screen initialization, mode switching, surface allocation, OpenGL setup with fallbacks, and frame composition (world view, HUD, terminals, maps). Central hub for all rendering output and display state.

## Core Responsibilities
- SDL2 window and renderer lifecycle management
- Screen mode configuration (resolution, fullscreen, acceleration, high-DPI)
- Rendering surface allocation and management (world, HUD, terminal, intro, map buffers)
- OpenGL context creation with multisampling and depth/stencil negotiation
- Gamma correction table management and application
- Color table (CLUT) handling for 8-bit and 16/32-bit modes
- View/HUD/terminal/map rectangle layout calculations
- Frame composition: blitting world view, overlays, HUD, and UI elements
- Viewport and scissor configuration for OpenGL
- Mouse and coordinate system transformations
- Screen clearing and margin management

## External Dependencies

- **SDL2:** SDL_Window, SDL_Renderer, SDL_Texture, SDL_Surface, SDL_Rect, SDL_Color, rendering/format functions
- **OpenGL:** glMatrixMode, glViewport, glOrtho, glScissor, glColor4f, glBlendFunc, etc. (via OGL_Headers.h); GLEW on Windows
- **Game subsystems:** 
  - `world.h` (world_view, view_data, world_point3d)
  - `map.h` (screen_mode_data, get_interface_rectangle)
  - `render.h` (render_view, initialize_view_data)
  - `OGL_*.h` (OGL_Blitter, OGL_Faders, OGL_SetWindow, OGL_StartRun, OGL_ClearScreen, OGL_SwapBuffers, OGL_RenderCrosshairs, OGL_DrawHUD)
  - `shell_options.h` (shell_options.nogamma)
  - `preferences.h` (graphics_preferences, interface_bit_depth, bit_depth)
  - `Crosshairs.h` (Crosshairs_IsActive, Crosshairs_Render)
  - `computer_interface.h`, `overhead_map.h`, `motion_sensor.h`, `screen_drawing.h` (various render/state functions)
  - `lua_hud_script.h`, `HUDRenderer_Lua.h` (Lua_DrawHUD, LuaHUDRunning)
  - `Movie.h` (movie recording frame updates)
- **Defined elsewhere:** render_view (render.cpp), _render_overhead_map, _render_computer_interface, draw_interface, DisplayNetLoadingScreen, DisplayPosition, DisplayMessages, update_fps_display, get_application_name, alert_out_of_memory, OGL_IsActive, MainScreenIsOpenGL, etc.

# Source_Files/RenderOther/screen.h
## File Purpose
Header for the Aleph One game engine's screen and rendering management system. Defines the singleton `Screen` class for mode/viewport management and declares functions for HUD rendering, color management, gamma correction, screen effects, and script-accessible display elements.

## Core Responsibilities
- Screen mode detection, selection, and dimension queries
- Viewport and clipping rectangle management for 3D view, HUD, map, and terminal
- Color table management (world, visible, interface) and gamma correction
- Screen effects (teleport, extravision, tunnel vision)
- HUD rendering control and script-based HUD element management
- Fullscreen toggle and display mode changes
- Overhead map zoom and visibility control
- Screenshot/screen dump functionality
- Main screen rendering pipeline and surface/window management

## External Dependencies
- **SDL2**: `SDL.h` for window, surface, rect, pixel format, events
- **Standard library**: `<utility>`, `<vector>` for mode list and pairs
- **Internal types** (defined elsewhere):
  - `struct Rect` (forward declared; likely a simple rect type)
  - `struct screen_mode_data` (display mode config)
  - `struct color_table` (palette data)
- **OpenGL** (implied by `OpenGLViewPort()`, `openGL()` queries, scissor methods, but not explicitly included here)

# Source_Files/RenderOther/screen_definitions.h
## File Purpose
Defines resource ID base constants for all major UI/screen elements in the game. Establishes a naming convention where 8-bit picture IDs are base values, 16-bit versions add 10000, and 32-bit versions add 20000.

## Core Responsibilities
- Define base resource IDs for intro, menu, prologue, epilogue, credit, chapter, computer interface, panel, and final screens
- Document the bit-depth offset scheme (8-bit + 0, 16-bit + 10000, 32-bit + 20000)
- Provide a centralized enumeration for screen resource lookups across the rendering and UI subsystems

## External Dependencies
- `<stdio.h>` implicit (standard C header conventions)
- No other includes visible

# Source_Files/RenderOther/screen_drawing.cpp
## File Purpose
Low-level SDL2-based rendering implementation for the Aleph One game engine's UI and HUD. Handles drawing shapes, text (bitmap and TrueType), rectangles, lines, and polygons to SDL surfaces with clipping support. Manages interface colors, fonts, layout rectangles, and provides an abstraction layer for switching drawing targets.

## Core Responsibilities
- **Port abstraction**: Redirect drawing operations to different SDL surfaces (main screen, HUD buffer, terminal, intro, map, custom)
- **Shape rendering**: Blit sprite graphics with optional source clipping
- **Text rendering**: Render both SDL bitmap fonts and TrueType fonts with styling (bold, italic, underline), alignment, and word wrapping
- **Geometric primitives**: Fill/frame rectangles, draw thin/thick lines with clipping, fill convex polygons with Sutherland-Hodgman clipping
- **Interface resource management**: Store and vend hardcoded interface rectangles, colors, and font specifications
- **Clip rectangle management**: Enable/disable drawing clipping regions for all drawing operations

## External Dependencies
- **SDL2**: SDL_Surface, SDL_Rect, SDL_LockSurface, SDL_BlitSurface, SDL_FillRect, SDL_FreeSurface, SDL_MapRGB, SDL_GetRGB
- **SDL2_ttf**: TTF_RenderUTF8_Blended/Solid, TTF_RenderUNICODE_Blended/Solid, TTF_FontAscent, TTF_FontHeight
- **Game headers**: shape_descriptors.h (get_shape_surface), screen.h (MainScreenSurface, MainScreenUpdateRects), sdl_fonts.h (font_info classes), FontHandler.h (FontSpecifier)
- **Utility**: process_printable(), process_macroman() (text encoding), get_ttf() (font lookup), environment_preferences (smooth_text flag)

# Source_Files/RenderOther/screen_drawing.h
## File Purpose

Header file defining the screen drawing API for the Aleph One game engine. Declares functions and constants for rendering shapes, text, UI elements, and primitives to various drawing contexts. Manages a port-based drawing system for flexible rendering to screen, buffers, and specialized surfaces (HUD, terminal, map, intro).

## Core Responsibilities

- Define enumerated constants for interface rectangles (HUD elements, menu buttons, terminal screens), colors, fonts, and text justification flags
- Declare port/context management functions to set drawing destination (screen, gworld buffer, terminal, HUD, etc.)
- Declare shape and sprite rendering functions with support for clipping and centering
- Declare text rendering functions with measurement, wrapping, and justification options
- Declare screen manipulation functions (clear, scroll, fill, frame)
- Provide interface configuration queries (get rectangle bounds, color values, font objects)
- Declare low-level drawing primitives (polygon, line, rectangle rasterization)
- Provide inline wrapper helpers around `font_info` for text measurement and drawing

## External Dependencies

- **shape_descriptors.h** ΓÇö `shape_descriptor` type and collection/shape enumeration
- **sdl_fonts.h** ΓÇö `font_info`, `FontSpecifier` abstract font classes; font loading/unload
- **SDL2** ΓÇö `SDL_Surface`, `SDL_Rect`, `SDL_ttf.h` (implied by sdl_fonts.h dependency)
- **Defined elsewhere:** `rgb_color` (likely in color header), `world_point2d` (geometry header), `TextSpec` (font spec; used by sdl_fonts.h)

# Source_Files/RenderOther/screen_shared.h
## File Purpose
Shared header declaring global screen state, messaging, HUD overlays, and text rendering utilities for the Aleph One game engine. Provides interfaces between screen.cpp and screen_sdl.cpp for color management, on-screen messages, FPS counters, and Script HUD element rendering.

## Core Responsibilities
- Manage global color tables (world, interface, visible, uncorrected) with gamma correction
- Queue and expire on-screen messages (up to 7 concurrent with automatic expiration)
- Manage Script HUD custom overlays per player with icon and text support
- Calculate and display FPS with high-resolution timing and network latency metrics
- Render debug overlays (player position, scores, network loading status)
- Control screen modes, view parameters, and gamma levels
- Implement zoom controls for overhead map
- Manage visual effects (teleport folds, extravision, tunnel vision)
- Handle text rendering with font fallback to OpenGL or SDL2

## External Dependencies
- **SDL2:** `SDL_Surface`, `SDL_Rect`, `SDL_Color`, `SDL_FillRect()`, `SDL_CreateRGBSurfaceFrom()`, `SDL_FreeSurface()`
- **OpenGL (conditional):** `OGL_IsActive()`, `OGL_RenderText()`, `OGL_RenderTextCursor()`, `OGL_TextWidth()`, `OGL_Blitter`
- **Rendering:** `OGL_Render.h`, `OGL_Blitter.h`, `Image_Blitter.h`, `screen_drawing.h`
- **Game world:** `view_data`, `screen_mode_data` (defined elsewhere), `world_pixels`, `world_view`, `current_player`, `dynamic_world`, `current_player_index`
- **Networking:** `NetGetLatency()`, `NetGetStats()`, `game_is_networked`
- **Color/UI:** `computer_interface.h`, `fades.h`, `_get_interface_color()`, `FontSpecifier`, `font_info`
- **Console:** `Console::instance()`, `displayBuffer()`, `input_active()`, `cursor_position()`
- **Utilities:** `machine_tick_count()`, `std::chrono::high_resolution_clock`, `vsnprintf()`

# Source_Files/RenderOther/Shape_Blitter.cpp
## File Purpose
Implements `Shape_Blitter`, a utility class for rendering shape/bitmap data from the engine's shapes collection to 2D UI elements. Supports both OpenGL and SDL software rendering with transformations like scaling, rotation, tinting, and cropping.

## Core Responsibilities
- Load and cache shape surfaces from the shapes collection on-demand
- Manage surface memory (allocation, conversion to BGRA8888, deallocation)
- Rescale surfaces with automatic crop-rectangle adjustment
- Render shapes via OpenGL (with texture manager integration) or SDL (software blitter)
- Apply transformations: tinting (RGBA), rotation about center, cropping, mirroring
- Handle shape-type-specific rendering: walls (rotated), landscapes (vertically flipped), sprites, interface elements
- Track both original and scaled source dimensions for coordinate mapping

## External Dependencies
- **Shape system:** `get_shape_surface()`, `extended_get_shape_information()`, `extended_get_shape_bitmap_and_shading_table()`, `shapes_file_is_m1()` ΓÇö load shape data and metadata
- **OpenGL texture manager:** `TextureManager`, `View_GetLandscapeOptions()`, `OGL_RenderTexturedRect()`, OpenGL headers
- **SDL:** `SDL_Surface`, `SDL_ConvertSurfaceFormat()`, `SDL_BlitSurface()`, surface creation/manipulation
- **Rendering config:** `Wanting_sRGB`, `Using_sRGB`, OGL setup/teardown globals
- **Image utilities:** `rescale_surface()`, `rotate_surface()`, `tile_surface()` (from images.h)
- **Shape definitions:** `shape_descriptor`, `shape_information_data`, texture type enums from scotish_textures.h and interface.h

# Source_Files/RenderOther/Shape_Blitter.h
## File Purpose
Defines the `Shape_Blitter` class for rendering 2D bitmaps from the game's Shapes file format to screen surfaces. Supports both OpenGL and SDL rendering backends with visual effects (tinting, rotation, cropping) for UI and sprite rendering.

## Core Responsibilities
- Load shape bitmaps from Shapes file collections with support for texture categories (walls, landscapes, sprites, weapons, interface)
- Manage SDL surface buffers (original and scaled copies)
- Perform dimension queries (scaled and unscaled)
- Render shapes via OGL_Draw() or SDL_Draw() with optional tinting and rotation
- Apply source-region cropping via crop_rect before rendering
- Clean up allocated SDL resources on destruction

## External Dependencies
- `cseries.h` ΓÇö platform abstraction, SDL2 includes
- `Image_Blitter.h` ΓÇö parent/related class defining `Image_Rect` struct and SDL patterns
- `map.h` ΓÇö Shapes file format context
- SDL2 ΓÇö surface and rendering
- `<vector>`, `<set>` ΓÇö standard containers (included but usage likely in .cpp)

# Source_Files/RenderOther/TextLayoutHelper.cpp
## File Purpose
Implements a 2D rectangle reservation/layout system that prevents overlapping UI elements. Given a rectangle's horizontal bounds and a minimum vertical position, it finds the lowest non-overlapping vertical placement by consulting existing reservations.

## Core Responsibilities
- Track active rectangle reservations across horizontal space using sorted endpoints
- Compute vertical placement for new rectangles by detecting conflicts with existing ones
- Manage memory lifecycle of reservation metadata
- Support clearing all reservations for reuse

## External Dependencies
- `#include <set>` ΓÇô for `std::multiset`
- `#include <assert.h>` ΓÇô runtime assertions
- `#include <vector>` (via header) ΓÇô `std::vector`

# Source_Files/RenderOther/TextLayoutHelper.h
## File Purpose
Utility class for managing non-overlapping rectangular regions in a 2D layout. Used by the Marathon: Aleph One engine to position UI elements (likely text boxes or other render components) without collisions.

## Core Responsibilities
- Track horizontal and vertical boundaries of placed rectangles
- Query for the lowest available vertical position at a given horizontal range
- Manage and clear all active reservations
- Calculate adjusted placement positions to avoid overlaps

## External Dependencies
- `<vector>` (STL for storing ReservationEnd list)
- `std::vector`

# Source_Files/RenderOther/TextStrings.cpp
## File Purpose
Implements a hierarchical text string repository system that replaces MacOS STR# resources. Strings are indexed by ID and sub-index, with support for XML/MML parsing and UTF-8 to ASCII/Mac Roman conversion for localization.

## Core Responsibilities
- Store, retrieve, and delete strings in a two-level indexed repository (ID ΓåÆ Index ΓåÆ String)
- Parse string definitions from InfoTree (XML/MML) configuration
- Convert UTF-8 encoded input strings to ASCII/Mac Roman format
- Provide null-pointer safety and bounds checking on all lookups
- Maintain global string state throughout engine lifetime

## External Dependencies
- `<string>`, `<map>` ΓÇö C++ STL containers
- `cseries.h` ΓÇö platform-abstraction types (`uint8`, `uint16`, `uint32`, `int16`, `size_t`)
- `TextStrings.h` ΓÇö public interface declarations
- `InfoTree.h` ΓÇö XML/MML tree parsing
- `unicode_to_mac_roman()` ΓÇö defined elsewhere; converts Unicode code points to Mac Roman bytes

# Source_Files/RenderOther/TextStrings.h
## File Purpose
Header for a text string repository system that replaces legacy MacOS STR# resources. Provides a two-level lookup scheme (resource ID + index) for managing localized and game strings. Includes UTF-8 decoding and MML markup support.

## Core Responsibilities
- Store and retrieve C-strings organized by resource ID and index
- Add/delete individual strings or entire string sets
- Query string set presence and cardinality
- Decode UTF-8 strings to C format
- Parse and manage MML-formatted string data

## External Dependencies
- `#include <stddef.h>` ΓÇö for `size_t` type
- `InfoTree` class ΓÇö defined elsewhere; represents hierarchical MML configuration data
- Called by render/UI systems expecting string IDs to map to localized content

# Source_Files/RenderOther/ViewControl.cpp
## File Purpose
Manages view rendering parameters including field-of-view (FOV), visual effects (fold/static effects on teleport), on-screen display font, and landscape texture options. Provides configuration parsing via Marathon Markup Language (MML) with state reset capabilities.

## Core Responsibilities
- Accessor functions for view settings (map visibility, teleport effects)
- FOV value management with smooth interpolation toward target values
- On-screen font initialization, scaling, and retrieval based on HUD scale level
- Landscape texture options storage and retrieval per collection/frame
- MML parsing and resetting for view and landscape configurations
- Backup/restore mechanism for configuration values

## External Dependencies
- **includes:** `<vector>`, `<string.h>`, `cseries.h`, `world.h`, `SoundManager.h`, `shell.h`, `screen.h`, `ViewControl.h`, `InfoTree.h`, `preferences.h`
- **defined elsewhere:** `graphics_preferences` (global prefs), `get_screen_mode()`, `MainScreenLogicalHeight()`, `FontSpecifier`, `InfoTree::read_*()` methods, `GET_DESCRIPTOR_*()` macros, shape descriptor utilities

# Source_Files/RenderOther/ViewControl.h
## File Purpose
View control subsystem for the Aleph One game engine. Manages camera/viewport parameters including field-of-view, landscape rendering, teleport effects, and on-screen display font. All parameters are configurable via XML.

## Core Responsibilities
- Field-of-view (FOV) management: accessors and adjustments for normal, extravision, and tunnel-vision modes
- Landscape/sky rendering configuration per environment/texture
- Teleport visual effects control: fold effects, static distortion, interlevel transition effects
- On-screen display (HUD) font retrieval
- Overhead map state query
- XML configuration parsing and reset for view and landscape settings

## External Dependencies
- `#include "world.h"` ΓÇö provides `angle` type
- `#include "FontHandler.h"` ΓÇö provides `FontSpecifier` class
- `#include "shape_descriptors.h"` ΓÇö provides `shape_descriptor` type
- `InfoTree` class (forward-declared, defined elsewhere) for XML parsing
- Implementation must define all `View_*` functions and landscape lookup table


