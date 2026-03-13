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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `FpsCounter` | class | Tracks FPS using high-resolution clock; updates every 250ms with running average |
| `ScreenMessage` | struct | On-screen message with expiration time (machine ticks) and text buffer (256 bytes) |
| `ScriptHUDElement` | struct | Custom HUD overlay per player containing icon data, color, text, and SDL/OGL blitters |
| `screen_mode_data` | struct (external) | Screen resolution, bit depth, gamma level, HUD scale |
| `view_data` | struct (external) | Camera position, angles, FOV, overhead map state, terminal mode, render effects |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `world_color_table` | `color_table*` | global | Gamma-corrected color palette for world rendering |
| `visible_color_table` | `color_table*` | global | Currently displayed color palette |
| `interface_color_table` | `color_table*` | global | 8-bit color palette for UI/fades |
| `uncorrected_color_table` | `color_table*` | global | Pristine unmodified color palette |
| `world_view` | `view_data*` | global | Main camera/view state (should be static) |
| `screen_mode` | `screen_mode_data` | static | Current screen configuration |
| `fps_counter` | `FpsCounter` | static | FPS calculation state with 250ms update interval |
| `displaying_fps` | bool | global | Toggle to show FPS overlay |
| `ShowPosition` | bool | global | Debug flag: display player position/angles |
| `ShowScores` | bool | global | Debug flag: display multiplayer scores |
| `Messages[]` | `ScreenMessage[7]` | static | Circular buffer of on-screen messages |
| `MostRecentMessage` | int | static | Index of most recently added message |
| `ScriptHUDElements[][]` | `ScriptHUDElement[8][6]` | static | HUD overlays (players ├ù max elements) |
| `nonlocal_script_hud` | bool | static | Display remote player's HUD instead of local |
| `HUD_RenderRequest` | bool | static | Deferred HUD rendering request flag |
| `Term_RenderRequest` | bool | static | Deferred terminal rendering request flag |
| `bit_depth`, `interface_bit_depth` | short | global | Color depth (16-bit or 8-bit) |

## Key Functions / Methods

### FpsCounter::update
- **Signature:** `void update()`
- **Purpose:** Accumulate one frame's FPS sample; recalculate average every 250ms
- **Side effects:** Updates `sum_`, `count_`, `prev_`, `fps_`, `next_update_`
- **Notes:** Invokes high-resolution clock; averages reciprocal of frame time in seconds

### FpsCounter::get / ready
- **Signature:** `float get() const`; `bool ready() const`
- **Purpose:** Return current FPS average; `ready()` returns true once first update interval completes
- **Outputs:** FPS value (float) or ready flag (bool)

### reset_messages
- **Signature:** `void reset_messages()`
- **Purpose:** Clear all on-screen messages and Script HUD elements; reset state
- **Side effects:** Zeroes message expiration times, clears HUD text/icons, sets `nonlocal_script_hud = false`

### reset_screen
- **Signature:** `void reset_screen()`
- **Purpose:** Reset screen/view state when starting a new game
- **Side effects:** Resets overhead map, terminal mode, FOV, scales via `world_view`; calls `ResetFieldOfView()`

### ResetFieldOfView
- **Signature:** `void ResetFieldOfView()`
- **Purpose:** Restore FOV based on current player's extravision state
- **Side effects:** Updates `world_view->field_of_view`, `target_field_of_view`, `tunnel_vision_active`
- **Notes:** Handles both normal (90┬░) and extravision (120┬░) FOV

### zoom_overhead_map_in / out
- **Signature:** `bool zoom_overhead_map_in/out()`
- **Purpose:** Increase/decrease overhead map scale (1ΓÇô4 range); return success
- **Outputs:** true if scale was changed
- **Side effects:** Updates `world_view->overhead_map_scale`

### start_teleporting_effect / start_extravision_effect
- **Signature:** `void start_teleporting_effect(bool out)`; `void start_extravision_effect(bool out)`
- **Purpose:** Initiate visual effects for teleport (fold in/out) and extravision (FOV transition)
- **Inputs:** `out` = true for activation, false for deactivation
- **Side effects:** Calls `start_render_effect()` (teleport) or updates `target_field_of_view` (extravision)

### DisplayText / DisplayTextCursor / DisplayTextWidth
- **Signature:** `void DisplayText(short BaseX, short BaseY, const char *Text, unsigned char r, g, b)`; `void DisplayTextCursor(SDL_Surface *s, short BaseX, short BaseY, const char *Text, short Offset, ...)`; `uint16 DisplayTextWidth(const char *Text)`
- **Purpose:** Render text (with shadow) or text cursor, or measure text width in current font
- **Inputs:** Screen coordinates, text, RGB color (defaults 0xFFFFFF), cursor offset
- **Side effects:** Draws to `DisplayTextDest` (SDL surface); tries OpenGL first, falls back to SDL2
- **Calls:** `OGL_RenderText()`, `OGL_RenderTextCursor()`, `OGL_TextWidth()`, `draw_text()`, `text_width()`
- **Notes:** Text coordinates are relative to surface; cursor height/width derived from font metrics

### DisplayPosition / DisplayInputLine / DisplayMessages / DisplayScores / DisplayNetLoadingScreen
- **Signature:** `void DisplayPosition(SDL_Surface *s)`; `void DisplayInputLine(SDL_Surface *s)`; etc.
- **Purpose:** Render debug info, console input, message queue, player scores/latency, or network loading status
- **Inputs:** SDL surface target, implicit global state
- **Side effects:** Calls `DisplayText()` repeatedly; colors depend on network stats (Green < 150ms, Yellow < 350ms, Red else)
- **Calls:** `DisplayText()`, `DisplayTextCursor()`, `NetGetStats()`, `calculate_player_rankings()`, etc.
- **Notes:** Layout uses font line spacing and margins from `alephone::Screen::instance()->lua_text_margins`; scores show ping, jitter, errors in color-coded format

### screen_printf
- **Signature:** `void screen_printf(const char *format, ...)`
- **Purpose:** Enqueue a printf-style message to on-screen message buffer
- **Inputs:** Format string and variadic args
- **Side effects:** Advances `MostRecentMessage` (circular), sets expiration to current tick + 7 seconds
- **Calls:** `vsnprintf()`, `machine_tick_count()`
- **Notes:** Uses safe `vsnprintf()` to prevent overflow; max message length 256 bytes

### get_screen_mode / game_window_is_full_screen / change_gamma_level
- **Signature:** `screen_mode_data *get_screen_mode()`; `bool game_window_is_full_screen()`; `void change_gamma_level(short gamma_level)`
- **Purpose:** Accessor for screen mode, fullscreen check, apply gamma correction
- **Outputs:** Pointer to static `screen_mode`; fullscreen bool (true if HUD hidden)
- **Side effects:** Gamma changes update color tables and trigger screen mode change
- **Calls:** `gamma_correct_color_table()`, `change_screen_mode()`, `stop_fade()`, etc.

### SetTunnelVision / GetTunnelVision
- **Signature:** `bool SetTunnelVision(bool TunnelVisionOn)`; `bool GetTunnelVision()`
- **Purpose:** Toggle tunnel vision mode and activate/deactivate visual effect
- **Outputs:** Current tunnel vision state (bool)
- **Side effects:** Updates `world_view->tunnel_vision_active`; calls `start_tunnel_vision_effect()`

### Script HUD Functions
- **Signature:** `void SetScriptHUDText(int player, int idx, const char* text)`; `void SetScriptHUDColor(...)`; `bool SetScriptHUDIcon(...)`; `void SetScriptHUDSquare(...)`; etc.
- **Purpose:** Update custom HUD overlays per player (text, color, icon, solid square)
- **Inputs:** Player index (mod 8), element index (mod 6), text/icon/color data
- **Side effects:** Modifies `ScriptHUDElements[player][idx]`; icon parsing converts custom format to RGBA; creates SDL/OGL blitters
- **Notes:** Icon format is custom (color count, palette, 16├ù16 bitmap); modulo operations clamp out-of-range indices

### RequestDrawingHUD / RequestDrawingTerm
- **Signature:** `void RequestDrawingHUD()`; `void RequestDrawingTerm()`
- **Purpose:** Flag that HUD/terminal should be redrawn on next frame
- **Side effects:** Sets `HUD_RenderRequest` or `Term_RenderRequest = true`

## Control Flow Notes
This file is primarily **state and display management** invoked during the render phase:
- **Initialization:** `reset_screen()` called when entering/resuming a game
- **Per-frame update:** `update_fps_display()` updates FPS counter; `DisplayText*()` functions draw overlays
- **Message handling:** `screen_printf()` queues messages; `DisplayMessages()` renders expiring messages each frame
- **Visual effects:** `start_*_effect()` functions initiate animations; effect state persists in `world_view`
- **Script integration:** Lua can call Script HUD setters to create custom overlays; rendering defers to `DisplayMessages()`

## External Dependencies
- **SDL2:** `SDL_Surface`, `SDL_Rect`, `SDL_Color`, `SDL_FillRect()`, `SDL_CreateRGBSurfaceFrom()`, `SDL_FreeSurface()`
- **OpenGL (conditional):** `OGL_IsActive()`, `OGL_RenderText()`, `OGL_RenderTextCursor()`, `OGL_TextWidth()`, `OGL_Blitter`
- **Rendering:** `OGL_Render.h`, `OGL_Blitter.h`, `Image_Blitter.h`, `screen_drawing.h`
- **Game world:** `view_data`, `screen_mode_data` (defined elsewhere), `world_pixels`, `world_view`, `current_player`, `dynamic_world`, `current_player_index`
- **Networking:** `NetGetLatency()`, `NetGetStats()`, `game_is_networked`
- **Color/UI:** `computer_interface.h`, `fades.h`, `_get_interface_color()`, `FontSpecifier`, `font_info`
- **Console:** `Console::instance()`, `displayBuffer()`, `input_active()`, `cursor_position()`
- **Utilities:** `machine_tick_count()`, `std::chrono::high_resolution_clock`, `vsnprintf()`
