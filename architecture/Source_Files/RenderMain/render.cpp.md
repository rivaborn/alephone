# Source_Files/RenderMain/render.cpp

## File Purpose

This is the central orchestration module for the rendering pipeline. It initializes and updates camera/view state each frame, coordinates visibility determination, polygon sorting, object placement, and rasterization into a coherent render pass, handles visual effects (explosions, teleports, camera shakes), applies transfer modes (texture slides, wobbles, pulsates, fades), and renders the weapon/HUD layer. It also supports Marathon 1 exploration missions by checking unseen polygons in the background.

## Core Responsibilities

- Initialize view/camera data with perspective calculations (FOV, cone angles, screen-to-world scaling)
- Update view state each tick (yaw/pitch vectors, FOV transitions, vertical pitch scaling)
- Apply render effects (fold in/out teleports, explosion camera shake) and transfer mode animations
- Orchestrate the multi-stage rendering pipeline: visibility tree ΓåÆ polygon sorting ΓåÆ object placement ΓåÆ rasterization
- Route rendering to appropriate rasterizer backend (software, OpenGL classic, or shader-based)
- Render player weapons and HUD sprites in foreground
- Manage transfer modes for both polygons (floors, ceilings, walls) and sprites/rectangles with various effects (slides, wobbles, static, fades)
- Support Marathon 1 exploration missions by periodically checking player views against unexplored polygons
- Allocate and initialize render memory and subsystem objects

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `view_data` | struct | Camera/viewpoint state: position, orientation, FOV, screen dimensions, media/liquid info, effects |
| `rectangle_definition` | struct (external) | Textured rectangle for sprite rendering with clipping, mirroring, and transfer modes |
| `polygon_definition` | struct (external) | Polygon data for transfer mode application (floors/ceilings/walls) |
| `weapon_display_information` | struct (external) | Weapon sprite display data: collection, shape, position, frame info, transfer mode |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `RenderFlagList` | `vector<uint16>` | global | Per-element visibility/clip flags indexed by endpoint/line/polygon |
| `RenderVisTree` | `RenderVisTreeClass` | static | Builds visibility tree of which polygons are visible from others |
| `RenderSortPoly` | `RenderSortPolyClass` | static | Sorts visible polygons into depth order; accumulates clip windows |
| `RenderPlaceObjs` | `RenderPlaceObjsClass` | static | Builds ordered list of renderable objects from sorted polygons |
| `Render_Classic` | `RenderRasterizerClass` | static | Classic clipping and rasterization orchestrator |
| `Rasterizer_SW` | `Rasterizer_SW_Class` | static | Software (CPU) rasterizer backend |
| `Rasterizer_OGL` | `Rasterizer_OGL_Class` | static | OpenGL rasterizer backend |
| `Rasterizer_Shader` | `Rasterizer_Shader_Class` | static | Shader-based rasterizer backend |
| `Render_Shader` | `RenderRasterize_Shader` | static | Shader clipping and rasterization orchestrator |
| `explore_view` | `view_data` | static | Fixed view for M1 exploration polygon checking |
| `explore_tree` | `RenderVisTreeClass` | static | Visibility tree for M1 exploration |

## Key Functions / Methods

### allocate_render_memory
- **Signature:** `void allocate_render_memory(void)`
- **Purpose:** Initialize all render subsystem memory and objects at game startup
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Resizes render flag vector; initializes and resizes visibility tree, polygon sorter, object placer; sets up cross-pointers between subsystems
- **Calls:** `RenderFlagList.resize()`, `RenderVisTree.Resize()`, `RenderSortPoly.Resize()`, etc.
- **Notes:** Called once per map load; asserts that pointer and POINTER_DATA sizes match for hack in subsystems

### initialize_view_data
- **Signature:** `void initialize_view_data(struct view_data *view, bool ignore_preferences = false)`
- **Purpose:** Set up camera perspective parameters based on FOV, screen dimensions, and aspect ratio
- **Inputs:** `view` (camera struct to initialize), `ignore_preferences` (bypass OpenGL gluPerspective adjustment if true)
- **Outputs/Return:** None (modifies `view` in-place)
- **Side effects:** Calculates and stores half-cone angles, world-to-screen scaling, landscape yaw; moves view origin off polygon vertices to avoid ray-cast singularities; sets media boundary state
- **Calls:** `View_AdjustFOV()`, `atan()`, `get_polygon_data()`, `get_endpoint_data()`, `get_media_data()`
- **Notes:** Handles non-standard aspect ratios by adjusting cone angle; corrects for degenerate polygons with vertices under the view origin; distinguishes between under/over media boundaries for liquid rendering

### render_view
- **Signature:** `void render_view(struct view_data *view, struct bitmap_definition *software_render_dest)`
- **Purpose:** Main render-pass orchestrator; executes full visibility, sorting, placement, and rasterization pipeline
- **Inputs:** `view` (camera state), `software_render_dest` (target framebuffer for software rendering, ignored under OpenGL)
- **Outputs/Return:** None
- **Side effects:** Updates view data, builds visibility tree, sorts polygons, places objects, rasterizes entire scene; resets overhead map; dispatches to terminal/overhead map renderers as needed
- **Calls:** `update_view_data()`, `ResetOverheadMap()`, `RenderVisTree.build_render_tree()`, `RenderSortPoly.sort_render_tree()`, `RenderPlaceObjs.build_render_object_list()`, `RasPtr->Begin()`, `RenPtr->render_tree()`, `render_viewer_sprite_layer()`, `RasPtr->End()`, `render_overhead_map()`, `render_computer_interface()`
- **Notes:** Skips main render if overhead map is active and opaque; coordinates both software and OpenGL backends; separates weapon/sprite rendering into `render_viewer_sprite_layer()` unless rasterizer handles it internally

### update_view_data
- **Signature:** `static void update_view_data(struct view_data *view)`
- **Purpose:** Refresh camera orientation vectors and depth scaling each tick
- **Inputs:** `view` (camera state to update)
- **Outputs/Return:** None (modifies `view` in-place)
- **Side effects:** Adjusts FOV toward target; applies render effects (if active); recalculates pitch tangent, left/right cone edge vectors; insets view origin away from polygon vertices; updates media boundary check
- **Calls:** `View_AdjustFOV()`, `update_render_effect()`, `cosine_table[]`, `sine_table[]`
- **Notes:** Idiot-proofs media accessor against null pointers; handles edge case where adjacent polygon has a vertex under view origin

### update_render_effect
- **Signature:** `static void update_render_effect(struct view_data *view)`
- **Purpose:** Advance and apply visual effects (explosions, teleport folds) to camera parameters
- **Inputs:** `view` (camera state with active effect and phase)
- **Outputs/Return:** None (modifies `view` in-place)
- **Side effects:** Increments effect phase; scales world-to-screen values for fold in/out; calls `shake_view_origin()` for explosions; clears effect when phase exceeds period
- **Calls:** `shake_view_origin()`, `MAX()`
- **Notes:** Uses interpolated phase with heartbeat fraction for smooth animation; fold effects distort perspective (x-scale up, y-scale down); explosion uses sine-wave camera shake on three axes

### instantiate_rectangle_transfer_mode
- **Signature:** `void instantiate_rectangle_transfer_mode(view_data *view, rectangle_definition *rectangle, short transfer_mode, _fixed transfer_phase)`
- **Purpose:** Apply transfer-mode visual effects (invisibility, static, pulsating, fold, etc.) to a sprite/rectangle
- **Inputs:** `view` (for shading info), `rectangle` (sprite to modify), `transfer_mode` (effect type), `transfer_phase` (animation phase 0ΓÇôFIXED_ONE)
- **Outputs/Return:** None (modifies `rectangle` in-place)
- **Side effects:** Changes rectangle's `transfer_mode`, `transfer_data`, `x0`/`x1` (shrink for fold), `HorizScale`; fetches global shading table for tinting
- **Calls:** `get_global_shading_table()`
- **Notes:** Invisibility renders as tinted transfer unless infravision is active; fold modes shrink sprite toward center (xc); supports stage clipping for foreground models

### instantiate_polygon_transfer_mode
- **Signature:** `void instantiate_polygon_transfer_mode(struct view_data *view, struct polygon_definition *polygon, short transfer_mode, bool horizontal)`
- **Purpose:** Apply transfer-mode visual effects to floor/ceiling/wall polygons
- **Inputs:** `view` (for tick count), `polygon` (floor/ceiling/wall to modify), `transfer_mode` (effect type), `horizontal` (true for floor/ceiling, false for wall)
- **Outputs/Return:** None (modifies `polygon` in-place)
- **Side effects:** Modifies polygon's `origin` (translates texture) and `vector` (wobbles/pulsates wall); applies shift phase and wander phase animations to texture coordinates
- **Calls:** `isqrt()`, `cosine_table[]`, `sine_table[]`
- **Notes:** Slides use bit-shifted phase; wander combines three frequencies (cosine for x, sine for y); pulsate/wobble use triangle-wave phase; fast modes multiply phase by 2 or 15

### render_viewer_sprite_layer
- **Signature:** `static void render_viewer_sprite_layer(view_data *view, RasterizerClass *RasPtr)`
- **Purpose:** Render player weapons and equipment sprites in foreground
- **Inputs:** `view` (camera, for screen dimensions and shading), `RasPtr` (rasterizer backend to dispatch to)
- **Outputs/Return:** None
- **Side effects:** Fetches weapon display info loop; builds rectangle definitions; applies shape flips, transfer modes, and clipping; dispatches to rasterizer's `texture_rectangle()`
- **Calls:** `get_weapon_display_information()`, `extended_get_shape_information()`, `OGL_GetModelData()`, `extended_get_shape_bitmap_and_shading_table()`, `instantiate_rectangle_transfer_mode()`, `RasPtr->texture_rectangle()`
- **Notes:** Skips if `view->show_weapons_in_hand` is false (third-person mode); supports optional 3D model replacement; applies shape mirror flags on top of display flags; limits clipping window to screen bounds; calculates center (xc) for teleport shrink effects

### check_m1_exploration
- **Signature:** `void check_m1_exploration(void)`
- **Purpose:** Background support for Marathon 1 exploration missions; periodically render each player's view without rendering to screen to mark explored polygons
- **Inputs:** None (reads global exploration mission flag, player list, tick count)
- **Outputs/Return:** None
- **Side effects:** On first call, initializes static `explore_view` and `explore_tree` with fixed M1-compatible FOV/resolution; each tick, samples players' views staggered by `TICKS_PER_EXPLORE`, calls `explore_tree.build_render_tree()` which marks polygons as explored; temporarily swaps and restores render flag list
- **Calls:** `initialize_view_data()`, `explore_tree.Resize()`, `update_view_data()`, `explore_tree.build_render_tree()`
- **Notes:** One-time initialization with hardcoded 80┬░ FOV and 640├ù320 view; marks explored polygons by side effect of visibility tree building; preserves main render flags across exploration checks

### position_sprite_axis
- **Signature:** `void position_sprite_axis(short *x0, short *x1, short scale_width, short screen_width, short positioning_mode, _fixed position, bool flip, world_distance world_left, world_distance world_right)`
- **Purpose:** Calculate screen-space boundaries for one axis (horizontal or vertical) of a weapon sprite
- **Inputs:** `x0`, `x1` (pointers to output bounds), `scale_width` (height for horiz, width for vert), `screen_width`, `positioning_mode` (low/center/high), `position` (fixed-point 0ΓÇô1), `flip`, `world_left`, `world_right` (world-space extents)
- **Outputs/Return:** Sets `*x0` and `*x1`
- **Side effects:** None (pure computation)
- **Calls:** None
- **Notes:** Positioning modes: `_position_low` = origin near screen edge (0 = invisible, 1 = fully visible); `_position_center` = origin on screen (0 = off low edge, 1 = off high edge); `_position_high` = origin near far edge (mirrored behavior); flipping reverses world left/right before scaling

### shake_view_origin
- **Signature:** `static void shake_view_origin(struct view_data *view, world_distance delta)`
- **Purpose:** Apply 3D sine-wave camera shake for explosion effect
- **Inputs:** `view` (camera to shake), `delta` (magnitude of shake)
- **Outputs/Return:** None (modifies `view->origin` in-place if polygon boundary not crossed)
- **Side effects:** Adds sine-wave offsets on three axes with different phases; only applies if result stays within current polygon (checked via `find_line_crossed_leaving_polygon()`)
- **Calls:** `sine_table[]`, `NORMALIZE_ANGLE()`, `find_line_crossed_leaving_polygon()`
- **Notes:** Uses `view->tick_count` to phase three axes differently; each axis uses separate FULL_CIRCLE multiple (7├ù for x, 7├ù + 5 seconds for y, 7├ù + 7 seconds for z); conservative: reverts shake if it would move view into different polygon

## Control Flow Notes

**Initialization (once per map):**
1. `allocate_render_memory()` initializes all render subsystems and allocates cross-pointers

**Per-frame render pipeline (`render_view()`):**
1. `update_view_data()` ΓÇô recalculate camera orientation, apply FOV transitions, apply effects
2. Clear render flags and overhead map
3. If not in terminal mode:
   - `RenderVisTree.build_render_tree()` ΓÇô determine visible polygons from camera position
   - If not overhead map or map is translucent:
     - `RenderSortPoly.sort_render_tree()` ΓÇô depth-sort polygons
     - `RenderPlaceObjs.build_render_object_list()` ΓÇô collect and sort visible objects
     - Select rasterizer backend (software vs. OpenGL)
     - `RenPtr->render_tree()` ΓÇô rasterize all visible geometry
     - `render_viewer_sprite_layer()` ΓÇô rasterize weapons on top
   - `render_overhead_map()` ΓÇô if map is active
4. If in terminal mode, `render_computer_interface()`

**Background (Marathon 1 exploration):**
- `check_m1_exploration()` runs in parallel, periodically sampling player views to mark unexplored polygons without affecting main render output

## External Dependencies

- **Notable includes:**
  - `map.h` ΓÇô polygon, line, endpoint, side structures; map accessors
  - `render.h` ΓÇô view_data, render flag definitions
  - `lightsource.h` ΓÇô light intensity lookups
  - `media.h` ΓÇô media/liquid state
  - `weapons.h` ΓÇô weapon display info
  - `player.h` ΓÇô player location and data
  - `RenderVisTree.h`, `RenderSortPoly.h`, `RenderPlaceObjs.h`, `RenderRasterize.h` ΓÇô render subsystem classes
  - `Rasterizer_SW.h`, `Rasterizer_OGL.h`, `Rasterizer_Shader.h` ΓÇô rasterizer backends
  - `OGL_Render.h` ΓÇô OpenGL-specific rendering (conditional)
  - `dynamic_limits.h`, `AnimatedTextures.h`, `preferences.h`, `screen.h` ΓÇô configuration and utility

- **External symbols used:**
  - `map_polygons`, `map_endpoints`, `map_lines` ΓÇô map geometry (from map.h)
  - `players`, `dynamic_world` ΓÇô game state (from player.h, map.h)
  - `graphics_preferences` ΓÇô renderer backend selection (preferences)
  - `render_computer_interface()`, `render_overhead_map()` ΓÇô external renderers (screen.c)
  - `get_weapon_display_information()`, `extended_get_shape_information()`, `extended_get_shape_bitmap_and_shading_table()` ΓÇô shape/texture accessors (interface.h)
  - `OGL_GetModelData()` ΓÇô 3D model lookup (OGL_Render.h)
  - Trig tables: `cosine_table[]`, `sine_table[]`, `NORMALIZE_ANGLE()` ΓÇô fixed-point math
