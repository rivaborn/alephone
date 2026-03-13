# Subsystem Overview

## Purpose
RenderOther handles all 2D screen composition, UI rendering, and visual effects for the Aleph One game engine. It manages the HUD, overhead map, text rendering, screen fades, and 2D asset blitting across multiple rendering backends (SDL2 software, OpenGL, Lua scripting). This subsystem orchestrates the complete display pipeline after 3D world rendering, composing menus, interface elements, on-screen messages, and tactical overlays.

## Key Files
| File | Role |
|------|------|
| `screen.cpp/h` | Central display and window lifecycle; surface allocation; frame composition orchestration; OpenGL setup |
| `HUDRenderer.cpp/h` | Base HUD rendering class; weapon/ammo/shield display; interface state management |
| `HUDRenderer_OGL.cpp/h` | OpenGL-accelerated HUD implementation with motion sensor, text, and shape rendering |
| `HUDRenderer_SW.cpp/h` | Software (SDL-based) HUD implementation for CPU rendering path |
| `HUDRenderer_Lua.cpp/h` | Lua-scripted HUD support with blip tracking and masking modes |
| `game_window.cpp/h` | HUD buffering, weapon interface layouts, MML customization, motion sensor initialization |
| `screen_drawing.cpp/h` | SDL2 primitive rendering (shapes, text, polygons, lines, rectangles) and port abstraction |
| `OverheadMapRenderer.cpp/h` | Base overhead map (automap) system; world-to-screen transforms; entity and annotation rendering |
| `OverheadMap_OGL.cpp/h` | OpenGL automap implementation with geometry batching and cached rendering |
| `OverheadMap_SDL.cpp/h` | SDL software automap implementation |
| `motion_sensor.cpp/h` | Radar/motion sensor display with entity tracking, blip rendering, and network compass |
| `computer_interface.cpp/h` | Terminal/computer interface rendering, text compilation, input handling, serialization |
| `fades.cpp/h` | Screen fade effects, color tinting, damage feedback, gamma correction, MML configuration |
| `FontHandler.cpp/h` | Font specification, metric calculation, OpenGL texture atlas generation, dual-mode text rendering |
| `images.cpp/h` | PICT resource loading/decompression, color table management, picture scaling and tiling |
| `Image_Blitter.cpp/h` | 2D image rendering with scaling, cropping, tinting |
| `OGL_Blitter.cpp/h` | OpenGL texture tiling and efficient 2D image rendering via GPU textures |
| `Shape_Blitter.cpp/h` | Shapes collection bitmap rendering with transformations (scale, rotate, tint) for UI sprites |
| `ViewControl.cpp/h` | FOV/camera parameters, landscape/sky rendering config, teleport effects, on-screen font management |
| `ChaseCam.cpp/h` | Third-person behind-view camera; physics-based damping; wall collision detection |
| `OGL_LoadScreen.cpp/h` | Load screen display with progress bars and image scaling |
| `TextStrings.cpp/h` | Localized text string repository; UTF-8 conversion; MML parsing |
| `TextLayoutHelper.cpp/h` | Non-overlapping rectangle reservation for UI layout |
| `screen_definitions.h` | Resource ID constants for UI screens with bit-depth offset scheme |
| `screen_shared.h` | Global screen state (color tables, on-screen messages, FPS display, visual effects) |

## Core Responsibilities

- Initialize and manage the game window, screen mode detection, surface allocation, and OpenGL context creation with fallback to software rendering
- Render the heads-up display (HUD) each frame with support for multiple backend implementations (SDL software, OpenGL hardware, Lua scripting)
- Compose the complete display each frame by blitting the 3D world view, overlaying HUD, terminal interfaces, overhead map, and screen effects
- Manage image and shape resource loading from WAD/resource files with color table handling and format conversion
- Render 2D primitive geometry (shapes, text, rectangles, lines, polygons) with SDL2 or OpenGL, supporting clipping and alignment
- Maintain the overhead map (automap) system with flood-fill visibility algorithms for checkpoint maps and dual rendering paths
- Apply screen effects including fades, color tints, damage feedback, gamma correction, and teleport visual transitions
- Provide text rendering with TrueType font support, bitmap fonts, scaling, word-wrap, and multi-backend abstraction
- Manage viewport and scissor configuration, coordinate transformation, and port-based drawing abstraction for flexible render target redirection

## Key Interfaces & Data Flow

**Exposes to other subsystems:**
- Screen mode API: `alephone::Screen` singleton for dimension queries, fullscreen toggle, color management, gamma correction
- HUD rendering entry point: `update_game_window()`, `draw_interface()` for per-frame composition
- Rendering primitives: `_draw_screen_shape()`, `_draw_screen_text()`, `_fill_rect()`, `_frame_rect()` for UI drawing
- Texture/image loading: `pictures_file`, `get_picture_resource_from_images()`, `picture_to_surface()` for asset retrieval
- Overhead map rendering: `_render_overhead_map()` for tactical display
- Screen effects: `screen_fade()`, `full_fade()` for cinematics and damage feedback
- Font management: `FontSpecifier` class for text layout and rendering

**Consumes from other subsystems:**
- Game world state: `dynamic_world`, `static_world`, polygon/line/endpoint data, entity positions (from `GameWorld`)
- Player data: `current_player`, `current_player_index`, weapon/item arrays, health/oxygen state (from `GameWorld`)
- Rendering state: `render_view`, `view_data`, `world_view` for 3D camera parameters and view frustum (from `RenderMain`)
- Resource files: Wad/shape/sound/image collections via `FileHandler` and `LoadedResource` (from `Files`)
- Configuration: Graphics preferences, bit depth, MML customizations via `InfoTree` parsing (from `XML/MML`)
- Sound system: `SoundManager` for HUD button click audio (from `Sound`)
- Scripting: Lua callbacks `L_Call_HUDDraw()`, `L_Call_Terminal_Exit()` (from `Lua`)
- Network state: `game_is_networked`, compass/team data for multiplayer HUD (from `Network`)

## Runtime Role

**Initialization:** Called via `initialize_screen()` to allocate surface buffers (world view, HUD, terminal, intro), configure OpenGL if available, set up color tables, and establish the window.

**Per-frame rendering:** Each game frame after 3D world rendering completes, `update_game_window()` and `draw_interface()` compose the final display by (1) updating HUD element state based on player condition, (2) rendering motion sensor and weapon displays, (3) applying fades and color effects, (4) rendering terminals/menus if active, (5) drawing the overhead map if visible, and (6) swapping buffers or blitting surfaces to the SDL window.

**Shutdown:** Surface cleanup via `screen.cpp` destructor or explicit unload functions for OpenGL textures, fonts, and image resources.

## Notable Implementation Details

- **Multiple HUD backends:** Polymorphic `HUD_Class` base class with three implementationsΓÇö`HUDRenderer_SW` (SDL software rasterization), `HUDRenderer_OGL` (OpenGL immediate-mode rendering), and `HUDRenderer_Lua` (Lua-scripted custom HUD). Runtime selection via `MainScreenIsOpenGL()` flag and `LuaHUDRunning()` check.
- **Dual rendering paths for images:** `Image_Blitter` abstract base with `OGL_Blitter` subclass that tiles SDL surfaces into power-of-two GPU textures for efficient OpenGL rendering; `Shape_Blitter` similarly abstracts shape bitmap rendering between SDL and OpenGL via texture manager.
- **Overhead map visibility algorithms:** `OverheadMapRenderer` implements flood-fill cost function to generate limited-visibility checkpoint maps; maintains separate `automap_polygons` and `automap_lines` bitfields for incremental visibility updates.
- **Port-based drawing abstraction:** `screen_drawing.cpp` maintains a global "port" pointer that redirects all drawing operations to different SDL surfaces (main screen, HUD buffer, terminal, intro screen, map buffer, custom), enabling UI composition to off-screen surfaces before final blit.
- **Gamma correction and color tables:** `screen.cpp` manages three color lookup tables (`world_color_table`, `visible_color_table`, `interface_color_table`) with gamma correction applied; supports both 8-bit indexed and 16/32-bit truecolor modes with automatic format conversion.
- **Font resource lifecycle:** `FontHandler` caches SDL_ttf font objects and generates OpenGL texture atlases with display lists for efficient glyph rendering; tracks active fonts in static registry for safe cleanup on OpenGL context loss.
