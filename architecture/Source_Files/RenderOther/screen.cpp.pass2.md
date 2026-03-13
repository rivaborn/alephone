# Source_Files/RenderOther/screen.cpp - Enhanced Analysis

## Architectural Role

This file is the **display composition and output orchestration hub** in Aleph One's rendering pipeline. It bridges the gap between high-level game concepts (view, HUD, terminals, map) and low-level SDL2/OpenGL presentation; it doesn't render 3D or UI itself, but coordinates *when* and *where* RenderMain, HUDRenderer_Lua, overhead_map, and computer_interface produce pixels, then composites the results into a single drawable frame. Its role as a junction between GameWorld/RenderMain (upstream producers) and SDL2/OpenGL (downstream consumers) makes it critical to maintaining separation of concerns: 3D rendering stays in RenderMain, display management stays here.

## Key Cross-References

### Incoming (Dependents)
- **Main game loop (marathon2.cpp)** ΓåÆ calls `update_world()` which implicitly triggers frame rendering via screen callbacks
- **Shell/Interface** ΓåÆ calls `enter_screen()`/`exit_screen()` to switch between menu and gameplay display modes
- **Lua HUD scripting (lua_hud_script.h, HUDRenderer_Lua.cpp)** ΓåÆ queries/overrides `lua_view_rect`, `lua_map_rect`, `lua_term_rect`, `lua_text_margins`; calls `L_Call_HUDResize` on mode changes
- **Input subsystem (mouse.h)** ΓåÆ calls `window_to_screen()` to de-aspect-correct mouse coordinates
- **GameWorld view updates** ΓåÆ modifies `world_view->target_field_of_view` (via `start_tunnel_vision_effect`), `overhead_map_active`, `terminal_mode_active`
- **Preferences (graphics_preferences)** ΓåÆ read for screen mode, gamma, HUD scaling levels, fullscreen state
- **Movie recording (Movie.h)** ΓåÆ frame buffer updated post-render for recording output

### Outgoing (Dependencies)
- **RenderMain (render.cpp)** ΓåÆ calls `render_view(world_view, software_render_dest)` to fill world_pixels with 3D scene
- **OGL_Render (OGL_Render.h/cpp, OGL_Setup.h/cpp)** ΓåÆ negotiates GL context creation, calls `OGL_StartRun`/`OGL_StopRun`, `OGL_IsActive`, `OGL_SwapBuffers`, `OGL_DrawHUD`, `OGL_ClearScreen`, `glViewport`/`glScissor`
- **OGL_Blitter** ΓåÆ renders HUD_Buffer and Term_Buffer textures to screen via `OGL_Blitter::Draw()`
- **CSeries color tables (cscluts.h)** ΓåÆ reads `uncorrected_color_table`, `world_color_table`, `visible_color_table`, `interface_color_table` for CLUT-based rendering and gamma
- **Overhead map (overhead_map.h)** ΓåÆ calls `_render_overhead_map()` with computed `overhead_map_data` scale/origin
- **Computer interface (computer_interface.h)** ΓåÆ calls `_render_computer_interface()` to draw terminal screen into Term_Buffer
- **Crosshairs (Crosshairs.h)** ΓåÆ calls `Crosshairs_Render()` after world view rendered
- **Screen drawing (screen_drawing.h)** ΓåÆ uses port management macros to set rendering destination (_set_port_to_screen_window, _set_port_to_HUD, etc.)
- **SDL2** ΓåÆ `SDL_CreateWindow`, `SDL_CreateRenderer`, `SDL_Texture`, surface blitting, gamma ramp (implicit via hardware)

## Design Patterns & Rationale

### 1. **Singleton Pattern** (Screen::m_instance)
Screen is a stateless singleton accessor for global rendering state (main_screen, main_surface, world_pixels, color tables). Rationale: avoids passing rendering context through dozens of call sites; centralized initialization/teardown point.

### 2. **Layered Composition with Independent Buffers**
- World view rendered to `world_pixels` (via RenderMain)
- HUD rendered to `HUD_Buffer` (separate 640├ù160 buffer, independently scalable)
- Terminal rendered to `Term_Buffer` (variable size based on terminal_scale_level)
- Intro rendered to `Intro_Buffer` (fixed 640├ù480)
- Map rendered to `Map_Buffer` (scaled per preference)

This decoupling allows each subsystem (HUD script, terminal renderer, map renderer) to produce output without coordinating resolution or aspect ratio. Each buffer has its own color space (gamma-corrected or not).

### 3. **Gamma Correction as Cross-Cutting Concern**
`current_gamma_r/g/b[256]` tables applied uniformly via `apply_gamma()` to world_pixels and intro_buffer. This ensures consistent color tone across rendering backends and smooths palette fades. Lazy initialization (`default_gamma_inited`) defers table construction until first use.

### 4. **State-Change Detection Pattern** (need_mode_change)
Rather than proactively tracking mode changes, `need_mode_change()` is queried before each potential mode switch, comparing current vs. requested dimensions/depth/GL. Fallback chain on creation failure (multisampling off ΓåÆ 16-bit depth ΓåÆ software renderer) avoids hard errors on incompatible hardware.

### 5. **Double-Buffering for Async Rendering**
`world_pixels` (uncorrected) and `world_pixels_corrected` (gamma-applied) allow RenderMain to write asynchronously without blocking gamma application. Lazy allocation defers memory cost until rendering actually occurs.

### 6. **Lua Override Pattern**
Lua HUD can override `lua_view_rect`, `lua_map_rect`, `lua_term_rect` to customize layout without C++ recompilation. Checked each frame in `view_rect()`, `map_rect()`, `term_rect()` ΓÇô design accommodates scripted UI without coupling to Lua internals.

## Data Flow Through This File

1. **Initialization**: `Screen::Initialize()` ΓåÆ allocates color tables, creates Intro_Buffer, calls `change_screen_mode()` ΓåÆ SDL window created, OpenGL/software renderer initialized, world_pixels allocated
2. **Per-frame (in-game)**:
   - `update_frame()` checks if view/map/term rects changed size
   - Calls `render_view(world_view, software_render_dest)` ΓåÆ RenderMain fills world_pixels with 3D scene
   - Calls `apply_gamma()` ΓåÆ world_pixels_corrected updated
   - Calls `update_screen()` ΓåÆ world_pixels_corrected blitted to main_surface with scaling (quadruple_surface for low-res)
   - Overlays rendered: `OGL_DrawHUD()` or `DrawSurface(HUD_Buffer)`, `_render_overhead_map()` if active, `_render_computer_interface()` if active
   - Calls `OGL_SwapBuffers()` (GL) or `MainScreenUpdateRects()` (software) ΓåÆ SDL_RenderPresent
3. **Mode transitions**: `enter_screen()` updates in_game flag, initializes Lua HUD rects, calls `change_screen_mode()` for level-specific settings (e.g., fullscreen in SP)
4. **Gamma/color changes**: `animate_screen_clut()` updates current_gamma_r/g/b, next frame applies via `apply_gamma()`

## Learning Notes

### Idiomatic Patterns
- **Port/rendering context abstraction** (via _set_port_to_* macros) ΓÇô avoids explicit context threading, common in 1990s engines
- **Explicit double-buffering** (world_pixels + world_pixels_corrected) ΓÇô predates GPU abstraction; manual control over memory layout for software scaling
- **Color lookup table (CLUT) persistence** ΓÇô legacy 8-bit palette support embedded in modern 32-bit renderer; gamma applied post-render for palette modes
- **Fallback rendering chains** ΓÇô GL feature negotiation (multisampling ΓåÆ depth precision ΓåÆ software mode) reflects shipping on diverse hardware

### Modern Engines Differ
- Modern engines (Unreal, Unity) abstract rendering via command buffers; screen.cpp directly calls SDL/OGL
- Gamma correction typically done in shaders; this engine applies CPU-side lookup tables
- Layout calculation (view_rect, hud_rect) is explicit here; modern engines often use constraint/anchor systems
- Lua script overrides of HUD rects is unusual; most engines use hierarchical scene graphs

## Potential Issues

**1. Gamma application bottleneck** (apply_gamma):
Iterates all pixels of multiple surfaces per frame (world + intro). On high-res displays (1920├ù1080+) with OpenGL enabled, this CPU work is redundant since GPU could apply gamma via shader/framebuffer. Workaround: if GL active, could skip apply_gamma and encode gamma in OpenGL pipeline.

**2. Coordinate space confusion**:
Multiple overlapping coordinate systems (logical width/height, window width/height, pixel coordinates, aspect-corrected GL ortho). `window_to_screen()` and `scissor_screen_to_rect()` perform conversions; easy to apply wrong transform when adding features.

**3. Mode change fallback chain fragility**:
If `change_screen_mode()` fails at GL multisampling stage, it retries with 16-bit depth, then software. If software renderer also fails (unlikely but possible on headless systems), no graceful degradationΓÇöprocess may exit. A "safe mode" (320├ù240 software) could be hardcoded as final fallback.

**4. Lua rect validation gap**:
Lua scripts can set lua_view_rect, lua_map_rect, lua_term_rect to arbitrary values; no bounds checking in `view_rect()`, `map_rect()`, `term_rect()`. Malicious Lua could cause off-screen rendering or division-by-zero in aspect ratio calculations.

---

This file exemplifies a **rendering coordinator** role: it doesn't perform 3D rendering, physics, or UI logic, but orchestrates their outputs into a cohesive display. Its cross-cutting responsibilities (gamma, layout, backend selection) reflect the tradeoffs of supporting multiple rendering paths (software, GL, shaders) on diverse hardware.
