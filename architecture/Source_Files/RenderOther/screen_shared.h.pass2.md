# Source_Files/RenderOther/screen_shared.h - Enhanced Analysis

## Architectural Role

This file acts as the **display composition hub** between the GameWorld simulation and the final 2D screen output. It mediates four key responsibilities: **(1)** persistent color management (uncorrected ΓåÆ gamma-corrected ΓåÆ visible pipeline), **(2)** transient overlay rendering (on-screen messages with 7-second expiration, Script HUD overlays per player), **(3)** performance telemetry (FPS counter with windowed averaging), and **(4)** view parameter synchronization (tunnel vision, extravision, overhead map zoom). It's the primary bridge allowing Lua scripts and debug systems to inject custom overlays without coupling to rendering backends.

## Key Cross-References

### Incoming (who depends on this)
- **screen.cpp / screen_sdl.cpp**: Display backends invoke `DisplayText*()`, `DisplayMessages()`, `DisplayScores()`, `DisplayFPS()`, `DisplayPosition()` during frame composition
- **Lua/ScriptHUDElement**: Script system calls `SetScriptHUDText()`, `SetScriptHUDColor()`, `SetScriptHUDIcon()` to create custom per-player overlays
- **network_games.cpp**: Calls `calculate_player_rankings()` to populate latency/ping data for ranked player display
- **marathon2.cpp** (game loop): Calls `reset_messages()` on level load; calls `update_fps_display()` per frame
- **Console / Debug**: Reads `displaying_fps`, `ShowPosition`, `ShowScores` flags; calls `screen_printf()` for on-screen debug output

### Outgoing (what this depends on)
- **OGL_Render.h**: Calls `OGL_RenderText()`, `OGL_RenderTextCursor()`, `OGL_TextWidth()` for OpenGL text rendering path
- **overhead_map.h**: `_render_overhead_map()` composes map rendering
- **computer_interface.h**: `_render_computer_interface()` renders terminal HUD
- **fades.h / render.h**: `stop_fade()`, `set_fade_effect()`, `change_screen_mode()` for visual transitions
- **Global state**: Reads/writes `world_view` (view_data), `world_pixels` (SDL_Surface), color tables, `screen_mode`

## Design Patterns & Rationale

**1. Cascading Color Table Pipeline**
- Three distinct color tables flow through gamma correction: `uncorrected_color_table` ΓåÆ `world_color_table` (gamma-corrected) ΓåÆ `visible_color_table` (with active fades applied)
- Decouples color correction (gamma) from fade effects (tints, static, dodge)
- Allows runtime gamma adjustment without regenerating base palette

**2. Circular Message Buffer with Tick-Based Expiration**
- `Messages[NumScreenMessages]` is a 7-slot circular queue indexed by `MostRecentMessage`
- Each message expires when `machine_tick_count()` exceeds `ExpirationTime` (default +7000 ticks Γëê 7 seconds)
- Machine ticks are deterministic and networking-friendly, unlike frame counts
- Avoids dynamic allocation; fixed buffer suitable for networked games where predictability matters

**3. Dual Rendering Path with Runtime Dispatch**
- Text rendering tries OpenGL first (`OGL_RenderText()`) if active; falls back to `draw_text()` + SDL2
- Separate blitter objects (`sdl_blitter`, `ogl_blitter`) stored per HUD element for compile-time #ifdef HAVE_OPENGL
- Allows graceful degradation on systems without OpenGL; no code duplication

**4. Per-Player Custom HUD as Array-of-Structures**
- `ScriptHUDElements[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS][MAXIMUM_NUMBER_OF_SCRIPT_HUD_ELEMENTS]` (8├ù6 grid)
- Each element independently holds icon, color, text, and pre-rendered blitter
- Flat array layout minimizes cache misses when rendering all players' HUDs
- Custom icon format (hex string) parsed once at assignment; blitters pre-baked for zero-copy rendering

**5. FpsCounter with 250ms Windowed Average**
- High-resolution clock (nanosecond precision) samples per-frame; accumulates reciprocal of frame time
- Updates published FPS every 250ms to reduce noise and UI flicker
- `ready()` flag prevents stale 0-fps display during first 250ms
- Idiomatic of early 2000s: explicit timing management before modern profilers

## Data Flow Through This File

**Message Display Path:**
1. `screen_printf(format, ...)` ΓåÆ `vsnprintf()` into circular `Messages[MostRecentMessage]` + sets expiration
2. `MostRecentMessage` advances (circular); new message overwrites oldest if buffer full
3. Per-frame: `DisplayMessages()` iterates buffer, renders non-expired entries via `DisplayText()`
4. `reset_messages()` zeroes all expiration times (signals immediate expire)

**Script HUD Path:**
1. `SetScriptHUDText/Color/Icon()` updates `ScriptHUDElements[player][idx]` fields
2. `SetScriptHUDIcon()` parses custom hex string format ΓåÆ `icon::parseicon()` ΓåÆ palette + bitmap
3. `icon::seticon()` converts bitmap to RGBA via palette lookup; creates SDL/OGL blitter
4. `DisplayMessages()` renders each active HUD element alongside text messages

**Color & Gamma Path:**
1. `change_gamma_level()` ΓåÆ `gamma_correct_color_table(uncorrected, world, gamma_level)` 
2. Copy `world_color_table` to `visible_color_table`
3. `change_screen_mode()` applies visible table to physical display

## Learning Notes

- **Early 2000s idioms**: Fixed-size circular buffers, machine ticks for determinism, exception-based parsing, explicit color cascade
- **Network-friendly design**: Tick-based expiration (not frame count) allows sync across varying frame rates
- **Legacy Mac compatibility**: Structure mimics old QuickDraw-era color management (separate uncorrected/corrected tables)
- **Custom binary formats**: Icon format is hand-rolled hex parser rather than PNG/image files (pre-embedded resource era)
- **Dual rendering path**: Reflects transition from software rasterizer ΓåÆ OpenGL; both backends still active

## Potential Issues

1. **Global state fragmentation**: Color tables, `world_view`, `world_pixels` are scattered globals; changes require coordination across subsystems
2. **Fixed message buffer**: Only 7 concurrent messages; rapid `screen_printf()` calls silently overwrite old messages without warning
3. **Icon parser brittle**: Custom hex format with exception-based error handling (`throw "end of string"`); invalid format silently fails during `SetScriptHUDIcon()`
4. **FpsCounter::ready() fragile**: Returns `fps_ != 0`; if frame time is exactly zero, `ready()` stays false indefinitely
5. **Unclamped modulo in Script HUD**: `player %= MAXIMUM_NUMBER_OF_NETWORK_PLAYERS` silently truncates out-of-range indices without validation
6. **DisplayText statics coupling**: `DisplayTextDest`, `DisplayTextFont`, `DisplayTextStyle` require external setup; no encapsulation or defaults
