# Source_Files/RenderOther/screen.h - Enhanced Analysis

## Architectural Role

This file is the **display composition layer** that bridges game world state (3D rendering output) with 2D screen elements (HUD, overhead map, terminal UI). It acts as the central **viewport manager** and **render orchestrator** for the frame, partitioning the screen into distinct regions and applying hardware-accelerated clipping. Every frame flows through `render_screen()`, making this the synchronization point between the 30 FPS game loop (GameWorld) and variable-rate rendering (RenderMain + OpenGL).

## Key Cross-References

### Incoming (who depends on this file)
- **GameWorld/marathon2.cpp**: Main loop calls `render_screen(ticks_elapsed)` each frame with elapsed time for animation
- **Lua scripts** (via script HUD functions): Call `SetScriptHUDText()`, `SetScriptHUDIcon()`, `SetScriptHUDColor()` to customize 6-element overlay per player
- **RenderMain/render.cpp**: Queries viewport rects (`view_rect()`, `hud_rect()`, etc.) to partition rendering targets
- **Input/mouse_sdl.cpp**: Calls `window_to_screen()` for mouselook coordinate conversion
- **RenderOther/overhead_map.cpp**: Queries `map_rect()` and `MainScreenPixelScale()` for overhead map dimensions
- **RenderOther/computer_interface.cpp**: Uses terminal viewport (`term_rect()`) and color palette functions

### Outgoing (what this file depends on)
- **SDL2** (SDL.h): Window creation/destruction, surface management, rect operations, pixel format descriptors
- **CSeries/cscluts.h**: Color palette construction (`build_color_table`), gamma correction (`change_gamma_level`)
- **GameWorld state** (implicit): Queries from game loop context (player state, map state) to compute viewports
- **RenderMain/render.h**: Calls to `render_screen()` coordinate with actual 3D rendering backend
- **CSeries/csmacros.h**: Bounds checking and flag manipulation for internal state

## Design Patterns & Rationale

**Singleton with Lazy Initialization**: `Screen::instance()` returns a static instance with boolean `m_initialized` flag. This pattern was common in early-2000s game engines for managing global subsystems (graphics, input, audio) before dependency injection became standard. Rationale: ensures single screen state across engine, simplifies API calls throughout codebase (no passing pointers).

**Viewport Partitioning via SDL_Rect Array**: Rather than a complex layout system, the file defines six overlapping rectangles (window, view, map, term, hud, OpenGLViewPort) that are computed once per mode change and cached. Rationale: avoids repeated computation in tight render loop; matches SDL's paradigm of clip rects and update regions from the 1990s.

**Dual Color Palette System**: Three separate `color_table` pointers (`world_color_table`, `visible_color_table`, `interface_color_table`) reflect **paletted graphics legacy** from 8-bit era where different screen regions used different color lookup tables. Modern code likely keeps them aliased, but the design preserves that architectural separation.

**Lua Script HUD as Extensibility Layer**: Rather than hard-coding HUD elements, `SetScriptHUDText/Icon/Color()` allow Lua mods to customize the overlay without recompiling. This is **Aleph One's key differentiator** as a moddable engine (post-2000 additions). The `MAXIMUM_NUMBER_OF_SCRIPT_HUD_ELEMENTS = 6` limit is a practical compromise between flexibility and fixed-size allocation.

## Data Flow Through This File

```
Game Loop (tick)
    Γåô
render_screen(ticks_elapsed)
    Γö£ΓöÇ Queries player viewport position ΓåÆ viewport_rect
    Γö£ΓöÇ Computes view_rect, hud_rect, map_rect, term_rect
    Γö£ΓöÇ Dispatches to RenderMain (3D rendering)
    Γö£ΓöÇ Applies color palette (visible_color_table)
    Γö£ΓöÇ Overlays script HUD elements (SetScriptHUDText results)
    Γö£ΓöÇ Renders overhead map (if active)
    Γö£ΓöÇ Renders terminal interface (if active)
    ΓööΓöÇ Flushes to screen (MainScreenSwap)

Display Mode Changes:
    change_screen_mode(mode_data)
        Γö£ΓöÇ Reinits SDL window/surface
        Γö£ΓöÇ Recalculates all viewport rects
        Γö£ΓöÇ Marks m_initialized = true
        ΓööΓöÇ Triggers redraw

Screen Effects (teleport, extravision):
    start_teleporting_effect() / start_extravision_effect()
        ΓåÆ State persisted in render loop
        ΓåÆ Applied via render_screen() fade/tint overlay
```

Key state: **`m_viewport_rect` and `m_ortho_rect`** are precomputed per mode and used by OpenGL scissor/viewport calls to clip rendering without per-frame recalculation.

## Learning Notes

- **Paletted Graphics Archaeology**: The three color tables and gamma correction functions reveal this engine descended from classic Mac Marathon (16-bit paletted graphics era). Modern ports keep the API for compatibility but likely alias tables to same palette.
  
- **Lua as Mod API**: Unlike modern engines that use C# or Rust for scripting, Aleph One exposes in-game HUD customization via LuaΓÇöevidences of a time when engines prioritized **lightweight scripting** over type safety. The 6-element limit for script HUD is "good enough" design for typical mods.

- **Software Rasterizer Still Present**: Methods like `scissor_screen_to_rect()` suggest software rendering path still exists alongside OpenGL. This was necessary for 2000s-era porting (not all platforms had GPU support), and the abstraction persists even if most users run OpenGL.

- **Frame Composition Over Post-Processing**: Rather than rendering to a single framebuffer and compositing in shaders (modern), this engine renders 3D view, then layers 2D UI directly onto screen. This is simpler but less flexible for effects like UI blur or dynamic scaling.

## Potential Issues

1. **Linear Mode Search**: `FindMode()` iterates `m_modes` linearlyΓÇöfine for 10ΓÇô20 modes, but O(n) lookup is inefficient if mode list grows.

2. **Color Palette Conflicts**: Three separate `color_table*` globals can be written to by different subsystems (gameplay vs. UI effects). No synchronization mechanism visible; could cause race conditions in multithreaded rendering.

3. **Viewport Rect Recomputation on Mode Change**: All six rects are recalculated togetherΓÇöif a single rect's formula changes (e.g., HUD resize), all six must be touched, risking cascading bugs.

4. **Script HUD Non-Local Flag**: `IsScriptHUDNonlocal()` / `SetScriptHUDNonlocal()` suggests script HUD rendering can be deferred to another context (networking?), but the actual non-local rendering code is not visible in this headerΓÇöcreates hidden coupling.

5. **No Viewport Validation**: `window_rect()` etc. return raw SDL_Rects without bounds checking. If rendering code passes invalid rects to `bound_screen_to_rect()`, undefined behavior may result.
