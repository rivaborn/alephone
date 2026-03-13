# Source_Files/RenderMain/render.h

## File Purpose
Central render system header defining the camera/view state structure (`view_data`), visibility/render flags for world geometry, and function prototypes for rendering initialization, frame rendering, and visual effects. Serves as the interface between the game engine and rendering backends (software and OpenGL).

## Core Responsibilities
- Define the `view_data` structure encapsulating all view parameters: camera position, orientation, FOV, screen dimensions, and effect state
- Declare render flag macros and bit definitions for marking polygon/endpoint/side visibility during traversal
- Export memory allocation and initialization for the rendering system
- Provide entry points for main view rendering, effect triggering, overhead map, and UI rendering
- Define render effects (fold-in/out, explosion) and shading modes (normal, infravision)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `view_data` | struct | Complete view state: camera position (origin, yaw/pitch/roll), FOV (current + target), screen metrics, effect phase, media/terminal state, weapons display flag, tunnel vision flag, landscape parameters |
| `definition_header` | struct | Minimal header for render objects: tag identifier and left/right clip bounds |
| render flag enum | enum | Bit definitions (_polygon_is_visible_bit, _endpoint_has_been_visited_bit, _endpoint_is_visible_bit, _side_is_visible_bit, _line_has_clip_data_bit, _endpoint_has_clip_data_bit, _endpoint_has_been_transformed_bit) |
| render effects enum | enum | _render_effect_fold_in, _render_effect_fold_out, _render_effect_explosion |
| shading modes enum | enum | _shading_normal (to black), _shading_infravision (false color) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `RenderFlagList` | `vector<uint16>` | global | Dynamic buffer storing render visibility/state flags for endpoints, lines, sides, polygons per frame |
| `render_flags` | macro (expands to `RenderFlagList.data()`) | global | Convenience macro providing array access to render flag buffer |

## Key Functions / Methods

### allocate_render_memory
- Signature: `void allocate_render_memory(void)`
- Purpose: Allocate and initialize dynamic buffers for rendering (e.g., RenderFlagList)
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates global render state buffers
- Calls: Not inferable from this file
- Notes: Called during engine initialization

### initialize_view_data
- Signature: `void initialize_view_data(struct view_data *view, bool ignore_preferences = false)`
- Purpose: Initialize or reset a view_data structure with sensible defaults for screen dimensions, FOV, and camera state
- Inputs: Pointer to view_data to initialize; optional flag to bypass user preferences
- Outputs/Return: Modifies view in-place
- Side effects: Sets screen_width, screen_height, FOV, and other cached projection parameters
- Calls: Likely calls ViewControl accessors (not visible here)
- Notes: Must be called before render_view; handles standard_screen_width ΓåÆ actual screen_width projection

### render_view
- Signature: `void render_view(struct view_data *view, struct bitmap_definition *software_render_dest)`
- Purpose: Render a single frame from the given view state to a bitmap destination (ignored under OpenGL)
- Inputs: Initialized view_data with camera position/orientation/FOV; destination bitmap (used for software rendering only)
- Outputs/Return: None (writes to framebuffer or OpenGL context)
- Side effects: Traverses world geometry, culls polygons, applies effects and transfer modes, updates tick counts
- Calls: Calls instantiate_rectangle_transfer_mode, instantiate_polygon_transfer_mode, and likely low-level rasterizers
- Notes: Core render dispatch; tick_count increments for effect animation; this is the main per-frame rendering entry point

### start_render_effect
- Signature: `void start_render_effect(struct view_data *view, short effect)`
- Purpose: Initiate a visual effect (fold-in, fold-out, explosion) on the next render frame
- Inputs: View to apply effect to; effect type from render effects enum
- Outputs/Return: None
- Side effects: Sets viewΓåÆeffect and resets viewΓåÆeffect_phase to 0
- Calls: Not inferable from this file
- Notes: Effects animate over multiple frames by incrementing effect_phase each render

### render_overhead_map
- Signature: `void render_overhead_map(struct view_data *view)`
- Purpose: Render the minimap/overhead view overlay
- Inputs: Current view_data for camera state
- Outputs/Return: None (renders to UI layer)
- Side effects: Queries viewΓåÆoverhead_map_active and overhead_map_scale
- Calls: Not inferable from this file
- Notes: Defined in SCREEN.C; only active if overhead_map_active is true

### render_computer_interface
- Signature: `void render_computer_interface(struct view_data *view)`
- Purpose: Render terminal/UI overlay (computer interface elements)
- Inputs: Current view_data for context
- Outputs/Return: None (renders to UI layer)
- Side effects: Queries viewΓåÆterminal_mode_active
- Calls: Not inferable from this file
- Notes: Defined in SCREEN.C; only active if terminal_mode_active is true

### instantiate_rectangle_transfer_mode / instantiate_polygon_transfer_mode
- Signature: `void instantiate_rectangle_transfer_mode(view_data *view, rectangle_definition *rectangle, short transfer_mode, _fixed transfer_phase)` and similar for polygon
- Purpose: Apply transfer mode (blending, tinting, static, landscape) to a single sprite or polygon during rendering
- Inputs: View, geometry definition, transfer mode type, phase (for animated transfers)
- Outputs/Return: None (modifies rendering state)
- Side effects: Prepares shading tables, applies palette tinting, or generates static pattern
- Calls: Not inferable from this file
- Notes: Helpers for render_view to handle transfer modes; separated for modular effect rendering

### check_m1_exploration
- Signature: `void check_m1_exploration(void)`
- Purpose: Check or update Marathon 1 exploration state (compatibility feature)
- Inputs: None (queries global game state)
- Outputs/Return: None
- Side effects: May update global exploration flags
- Calls: Not inferable from this file
- Notes: Marathon 1 compatibility hook; called during render or update

### ResetOverheadMap
- Signature: `void ResetOverheadMap()`
- Purpose: Clear or reinitialize the overhead map display
- Inputs: None
- Outputs/Return: None
- Side effects: Clears overhead map state (defined in overhead_map.cpp)
- Calls: Not inferable from this file
- Notes: Defined in overhead_map.cpp; typically called on level load or manual toggle

## Control Flow Notes
This file is the **render initialization and dispatch layer**:
1. **Startup**: `allocate_render_memory()` allocates flag buffers; `initialize_view_data()` sets up view parameters.
2. **Per-frame**: `render_view()` is called once per frame with the current player view; it uses the render flag buffer to mark geometry visibility.
3. **Effects**: `start_render_effect()` queues effects; `render_view()` applies them based on effect_phase.
4. **UI overlay**: `render_overhead_map()` and `render_computer_interface()` render UI on top of world geometry.
5. **Geometry processing**: Transfer modes are instantiated per-object by helper functions during render_view traversal.

The tick_count in view_data drives all time-dependent rendering (effects, animated textures, blinking).

## External Dependencies
- **world.h**: `world_point3d`, `world_distance`, `angle`, `world_vector3d`, `fixed_angle`, trigonometric types
- **textures.h**: `bitmap_definition` (texture/sprite data)
- **scottish_textures.h**: `rectangle_definition` (sprites), `polygon_definition` (walls/floors/ceilings)
- **ViewControl.h**: `View_FOV_Normal()`, `View_FOV_ExtraVision()`, `View_FOV_TunnelVision()` (FOV accessors); effect and effect-phase control functions
- Implicit OpenGL context (for rendering backend; not included in this header)
