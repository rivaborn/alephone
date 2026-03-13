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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `bitmap_definition_buffer` | struct/wrapper | Converts SDL_Surface into bitmap_definition for rendering |
| `screen_mode_data` | struct | Screen configuration (resolution, fullscreen, acceleration, gamma, scaling) |
| `view_data` | struct | Camera/viewport state (FOV, position, pitch/yaw, screen dimensions) |
| `color_table` | struct | 256-entry color palette for 8-bit modes or gamma lookup tables |
| `SDL_Rect` | struct | Rectangle for window/viewport/surface regions |
| `OGL_Blitter` | class | OpenGL texture blitter for HUD/terminal surfaces (ifdef HAVE_OPENGL) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `main_screen` | SDL_Window* | static | Active SDL window handle |
| `main_surface` | SDL_Surface* | static | Main logical display surface (rendered content) |
| `main_render` | SDL_Renderer* | static | SDL2 software renderer (non-GL path) |
| `main_texture` | SDL_Texture* | static | Texture backing for software renderer updates |
| `world_pixels` | SDL_Surface* | global | Raw rendering buffer for world view |
| `world_pixels_corrected` | SDL_Surface* | global | Gamma-corrected copy of world_pixels |
| `HUD_Buffer` | SDL_Surface* | global | HUD rendering surface (640├ù160) |
| `Term_Buffer` | SDL_Surface* | global | Terminal screen rendering surface |
| `Intro_Buffer` | SDL_Surface* | global | Menu/intro/credits screen surface (640├ù480) |
| `Map_Buffer` | SDL_Surface* | global | Overhead map rendering surface |
| `current_gamma_r/g/b[256]` | uint16[] | global | Active gamma correction tables |
| `default_gamma_r/g/b[256]` | uint16[] | global | Linear (identity) gamma tables |
| `uncorrected_color_table` | color_table* | global | Raw palette or color data (unchanged by gamma) |
| `world_color_table` | color_table* | global | Gamma-corrected world palette |
| `visible_color_table` | color_table* | global | Currently active palette for rendering |
| `interface_color_table` | color_table* | global | Palette for UI/HUD (cached for quick updates) |
| `in_game` | static bool | static | True if in gameplay mode; false if in menus (affects scaling/layout) |
| `PrevFullscreen` | static bool | static | Previous fullscreen state to detect transitions |
| `failed_multisamples` | static int | static | MSAA sample count that failed, to prevent retries |
| `passed_shader` | static bool | static | Track if GL_ARB_shader extensions passed at startup |
| `default_gamma_inited` | static bool | static | Whether gamma tables have been populated |
| `using_default_gamma` | static bool | static | True if current gamma = linear (no correction) |
| `software_render_dest` | bitmap_definition_buffer | static | View of world_pixels as bitmap for software rendering |
| `pixel_format_16` | SDL_PixelFormat | static | 16-bit RGB565 pixel format descriptor |
| `pixel_format_32` | SDL_PixelFormat | static | 32-bit ARGB8888 pixel format descriptor |
| `Term_Blitter` | OGL_Blitter | static | OpenGL blitter for terminal rendering |
| `Intro_Blitter` | OGL_Blitter | static | OpenGL blitter for intro screen rendering |
| `m_instance` | Screen (static member) | global | Singleton Screen instance |

## Key Functions / Methods

### Screen::Initialize
- **Signature:** `void Initialize(screen_mode_data* mode)`
- **Purpose:** One-time initialization of the screen subsystem; creates pixel format descriptors, allocates color tables, initializes view_data, builds list of available screen modes, validates preferences, and enters change_screen_mode.
- **Inputs:** `screen_mode_data* mode` ΓÇö requested screen mode
- **Outputs/Return:** None
- **Side effects:** Allocates global color tables, SDL surfaces (Intro_Buffer, Intro_Buffer_corrected), populates m_modes list, calls change_screen_mode, sets screen_initialized flag
- **Calls:** `SDL_AllocFormat`, `SDL_CreateRGBSurface`, `malloc`, `change_screen_mode`, `FindMode`, `write_preferences`
- **Notes:** Called once per app lifetime; on reinit unloads collections and frees old surfaces; uses lazy lazy initialization pattern for world_pixels

### Screen::change_screen_mode
- **Signature:** `static void change_screen_mode(int width, int height, int depth, bool nogl, bool force_menu, bool force_resize_hud=false)`
- **Purpose:** Core mode-switching logic; creates/destroys SDL window, sets up OpenGL or software renderer, configures logical size, handles multisampling fallback, shader validation, and notifies about render subsystem (OGL_StartRun/StopRun).
- **Inputs:** width, height (logical/window dims), depth (bit depth), nogl (disable GL), force_menu (640├ù480), force_resize_hud
- **Outputs/Return:** None (modifies global main_screen, main_render, main_texture, main_surface)
- **Side effects:** Destroys/recreates main_screen window, renderer, context; sets OpenGL attributes; logs fallback attempts (multisamplingΓåÆ16-bit depthΓåÆsoftware); calls ReloadViewContext on Windows/macOS
- **Calls:** `need_mode_change`, `SDL_CreateWindow`, `SDL_GL_SetAttribute`, `OGL_CheckExtension`, `glewInit`, `SDL_CreateRenderer`, `SDL_CreateTexture`, `reallocate_world_pixels`, `OGL_StartRun`
- **Notes:** Retries with reduced GL features if initial create fails (multisampling off ΓåÆ 16-bit depth ΓåÆ software renderer); long fallback chain on Windows for GL compatibility; includes commented-out legacy code paths

### need_mode_change
- **Signature:** `static bool need_mode_change(int window_width, int window_height, int log_width, int log_height, int depth, bool nogl)`
- **Purpose:** Check if current mode matches requested; if not, update window/GL settings and return false so caller proceeds with recreation.
- **Inputs:** Requested window/logical dimensions, depth, nogl flag
- **Outputs/Return:** `bool` ΓÇö true if already matches (no change needed), false if mismatch (change required)
- **Side effects:** Calls SDL_SetWindowFullscreen, SDL_SetWindowSize, SDL_RenderSetLogicalSize, SDL_GL_SetSwapInterval, SDL_SetWindowTitle; may destroy/recreate main_surface or main_texture
- **Calls:** `SDL_GetWindowFlags`, `SDL_GetWindowSize`, `SDL_GetRendererOutputSize`, `SDL_QueryTexture`, `SDL_GL_GetAttribute`, `SDL_RenderGetLogicalSize`
- **Notes:** Returns early with false if mismatch detected; fullscreen/window switching and vsync/MSAA changes checked here

### update_frame (unnamed main loop function in file)
- **Signature:** Not formally named; invoked from game loop to render a frame
- **Purpose:** Main per-frame orchestration: updates view/map/term rects, reallocates buffers if mode changed, renders world, overlays HUD/terminal/map, handles Lua HUD, composites to screen via OpenGL or software update.
- **Inputs:** Implicit (global state: world_view, screen_mode, HUD/Term/Map buffers)
- **Outputs/Return:** None
- **Side effects:** Clears screen/margins, calls render_view, updates HUD/map/terminal surfaces, calls OGL_SwapBuffers or MainScreenUpdateRects, requests frame recording
- **Calls:** `clear_screen_margin`, `reallocate_world_pixels`, `reallocate_map_pixels`, `render_view`, `Crosshairs_Render`, `update_fps_display`, `DisplayPosition`, `DisplayMessages`, `OGL_DrawHUD`, `Lua_DrawHUD`, `DrawSurface`, `darken_world_window`
- **Notes:** Large function (~350 lines); handles both OpenGL and software paths; tracks previous rect sizes to detect changes

### render_view
- **Signature:** `void render_view(struct view_data *view, struct bitmap_definition *software_render_dest)`
- **Purpose:** Render the 3D world into software_render_dest (or to OpenGL if OGL_IsActive); core rendering engine entry point.
- **Inputs:** `view_data* view` (camera/viewport), `bitmap_definition* software_render_dest` (output buffer, ignored in GL mode)
- **Outputs/Return:** None
- **Side effects:** Fills software_render_dest surface with rendered 3D scene; may invoke OGL rendering pipeline
- **Calls:** Defined elsewhere (render.cpp); uses interpolate_world_view internally
- **Notes:** Called once per frame; integrates with render.cpp for actual geometry/lighting

### apply_gamma
- **Signature:** `static void apply_gamma(SDL_Surface *src, SDL_Surface *dst)`
- **Purpose:** Pixel-by-pixel gamma correction; applies current_gamma_r/g/b lookup tables to each RGB component.
- **Inputs:** `src`, `dst` ΓÇö source and destination surfaces (must be same dimensions, 16 or 32-bit)
- **Outputs/Return:** None; modifies dst
- **Side effects:** Locks dst if required; iterates all pixels and rewrites with gamma-corrected values
- **Calls:** `SDL_MUSTLOCK`, `SDL_LockSurface`, `SDL_UnlockSurface`
- **Notes:** Handles both 16-bit (RGB565) and 32-bit (ARGB8888); extracts R/G/B via masks and shifts; supports little-endian and big-endian platforms via pixel format descriptors

### update_screen
- **Signature:** `static void update_screen(SDL_Rect &source, SDL_Rect &destination, bool hi_rez, bool every_other_line)`
- **Purpose:** Blit world_pixels to main_surface, optionally quadrupling for low-res modes or applying gamma correction.
- **Inputs:** source/destination rects, hi_rez (1x or 2x scale), every_other_line (interlace for overlay maps)
- **Outputs/Return:** None; modifies main_surface
- **Side effects:** Locks main_surface if needed; may create intermediary surface for format conversion
- **Calls:** `apply_gamma`, `SDL_BlitSurface`, `SDL_ConvertSurface`, `quadruple_surface`
- **Notes:** If not hi_rez, calls quadruple_surface template to 2├ù scale; if color format mismatch, converts via SDL_ConvertSurface

### quadruple_surface (template)
- **Signature:** `template <class T> static inline void quadruple_surface(const T *src, int src_pitch, T *dst, int dst_pitch, const SDL_Rect &dst_rect, bool every_other_line)`
- **Purpose:** 2├ù pixel scaling for low-res rendering; duplicates each pixel horizontally and vertically, optionally interleaving black scanlines for overlay maps.
- **Inputs:** src/dst pointers, pitches, destination rect, every_other_line flag
- **Outputs/Return:** None; modifies dst
- **Side effects:** Writes scaled pixels into dst; black_pixel used to clear alternate rows if overlay active
- **Calls:** `SDL_MapRGB` (once at start)
- **Notes:** Template instantiated for pixel8, pixel16, pixel32; handles overlay_active case to preserve transparency

### clear_screen / clear_screen_margin
- **Signature:** `void clear_screen(bool update)` / `void clear_screen_margin()`
- **Purpose:** Fill display with black; clear_screen_margin only clears borders around active view/map/term rects.
- **Inputs:** update (whether to call MainScreenSwap/UpdateRect)
- **Outputs/Return:** None
- **Side effects:** Fills main_surface or calls OGL_ClearScreen; optionally swaps GL buffers or updates texture
- **Calls:** `OGL_ClearScreen`, `SDL_FillRect`, `MainScreenUpdateRect`, `MainScreenSwap`
- **Notes:** clear_screen_margin is more efficient for partial updates

### enter_screen / exit_screen
- **Signature:** `void enter_screen(void)` / `void exit_screen(void)`
- **Purpose:** Transition into/out of gameplay mode; sets in_game flag, initializes Lua HUD rects, starts/stops OpenGL rendering, resets controller state.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets in_game=true/false; calls change_screen_mode for level mode; calls OGL_StartRun/OGL_StopRun; initializes lua_*_rect and lua_text_margins; calls L_Call_HUDResize; resets SDL modifier state
- **Calls:** `change_screen_mode`, `OGL_StartRun`, `OGL_StopRun`, `L_Call_HUDResize`, `set_overhead_map_status`, `set_terminal_status`
- **Notes:** enter_screen also clears world_view.effect; Lua clip/view/map/term/text rects are set for HUD layout

### Screen accessor methods
- **height/width/pixel_scale/window_height/window_width/hud/lua_hud/openGL/fifty_percent/seventyfive_percent:**
  - Simple getter methods; query screen_mode or call MainScreen*() functions
  - Return logical height/width, window size, pixel scale, HUD active, OpenGL active, scaling flags

### Screen layout methods
- **window_rect / view_rect / map_rect / term_rect / hud_rect:**
  - Compute SDL_Rect for each UI element based on logical/window size, HUD state, Lua overrides, scaling
  - Return rectangles in screen space
  - view_rect: clamps to 2:1 aspect for display window; respects lua_view_rect if Lua HUD active
  - hud_rect: 640-wide base, scaled to 2├ù at high res; positioned at bottom
  - term_rect: terminal screen, scaled based on term_scale_level preference

### bound_screen / bound_screen_to_rect / scissor_screen_to_rect / window_to_screen
- **Purpose:** OpenGL viewport/scissor configuration and coordinate transforms
- **Signature:** `void bound_screen(bool in_game)` / `void bound_screen_to_rect(SDL_Rect &r, bool in_game)` / `void scissor_screen_to_rect(SDL_Rect &r)` / `void window_to_screen(int &x, int &y)`
- **Side effects (GL paths):** `glViewport`, `glScissor`, `glMatrixMode`, `glLoadIdentity`, `glOrtho`; updates m_viewport_rect, m_ortho_rect
- **Notes:** Converts virtual coordinates to pixel/GL coordinates; scissor clips rendering to a rect; window_to_screen de-aspect-corrects mouse input

### MainScreen* utility functions
- `MainScreenVisible`, `MainScreenLogicalWidth/Height`, `MainScreenWindowWidth/Height`, `MainScreenPixelWidth/Height`, `MainScreenPixelScale`, `MainScreenIsOpenGL`, `MainScreenSwap`, `MainScreenCenterMouse`, `MainScreenSurface`, `MainScreenWindow`, `MainScreenUpdateRect`, `MainScreenUpdateRects`
  - Global entry points for querying/updating display state without direct access to static globals
  - `MainScreenUpdateRects` updates SDL texture and renders to screen via SDL_RenderPresent

### Color/gamma functions
- **initialize_gamma / build_direct_color_table / change_screen_clut / change_interface_clut / animate_screen_clut / assert_world_color_table:**
  - Manage gamma tables and color lookup tables
  - change_screen_clut applies gamma_correct_color_table
  - animate_screen_clut updates current_gamma_r/g/b for live fades
  - assert_world_color_table syncs palette to 8-bit surfaces if needed

### Render overhead map / terminal
- **render_overhead_map / render_computer_interface:**
  - Set up viewport/scissor, fill Map_Buffer or terminal surface, call _render functions
  - render_overhead_map configures overhead_map_data (scale, origin, dimensions) before calling _render_overhead_map

### Other utilities
- **start_tunnel_vision_effect:** Sets world_view->target_field_of_view based on NetAllowTunnelVision and current_player extravision
- **ReloadViewContext:** Calls OGL_StartRun if in gameplay with GL enabled (Windows/macOS workaround)
- **map_is_translucent:** Returns true if translucent map allowed and enabled in preferences + network
- **darken_world_window:** Draw dithered 50% black overlay (GL or software); masks inactive game window
- **validate_world_window / update_screen_window:** Request terminal redraw / sync interface color table
- **calculate_destination_frame:** Compute rect for world view at given zoom level
- **draw_intro_screen / DrawSurface:** Composite intro buffer or HUD/terminal to main_surface via GL Blitter or SDL blit

## Control Flow Notes

**Initialization sequence (once):**
1. Screen::Initialize() ΓåÆ allocates surfaces, color tables, sets up modes list, calls change_screen_mode
2. change_screen_mode() ΓåÆ creates SDL window, GL context or software renderer, allocates main_surface

**Per-frame (gameplay):**
1. update_frame() (implicit, called from main loop)
   - Checks if view/map/term rects changed size
   - Reallocates world_pixels/Map_Buffer if needed
   - Calls render_view() to fill world_pixels
   - Calls update_screen() to copy world_pixels ΓåÆ main_surface (with gamma, scaling)
   - Blits HUD/map/term overlays via DrawSurface or OGL_Blitter
   - Calls OGL_SwapBuffers or MainScreenUpdateRects to present

**Mode change (user-initiated or level load):**
1. enter_screen() / exit_screen() wrap level transitions
2. change_screen_mode() recreates window/renderer as needed, validates GL features
3. Lua HUD rects initialized in enter_screen

**Rendering paths:**
- **OpenGL:** render_view fills OGL pipeline; OGL_DrawHUD draws HUD; OGL_Blitter renders terminal/intro; OGL_SwapBuffers presents
- **Software:** render_view fills world_pixels; update_screen blits to main_surface with scaling/gamma; DrawSurface/MainScreenUpdateRects update SDL texture and present

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
