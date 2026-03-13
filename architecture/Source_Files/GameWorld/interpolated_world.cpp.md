# Source_Files/GameWorld/interpolated_world.cpp

## File Purpose

Provides high-frame-rate rendering (>30 FPS) by interpolating world state between fixed 30 FPS game ticks. Maintains double-buffered snapshots of game world (objects, polygons, sides, lines, ephemera, camera, weapons) and blends them each render frame based on elapsed time.

## Core Responsibilities

- Initialize and manage double-buffered world state snapshots
- Capture complete world state at each 30 FPS tick boundary
- Calculate per-frame interpolation progress (0ΓÇô1) based on machine tick counter
- Interpolate object positions with speed-based rejection (don't interpolate fast-moving projectiles)
- Interpolate camera rotation and position with angle wraparound handling
- Interpolate polygon/line geometry heights and side texture offsets
- Handle object/ephemera polygon crossings during interpolation
- Track projectile positions for contrail effect rendering
- Interpolate weapon display information (ammo count, shell positions)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| TickObjectData | struct | Snapshot: object position, polygon, flags, linked list pointer |
| TickPolygonData | struct | Snapshot: floor/ceiling heights, first object in polygon list |
| TickSideData | struct | Snapshot: side texture Y-offset coordinate |
| TickLineData | struct | Snapshot: line adjacent floor/ceiling heights |
| TickWorldView | struct | Snapshot: camera position, rotation (yaw/pitch), depth intensity |
| ContrailInfo | struct | Projectile location tracking for contrail interpolation |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| world_is_interpolated | bool | global | Interpolation currently active flag |
| start_machine_tick | uint64_t | static | Machine tick at frame start (for heartbeat calculation) |
| default_speed_limit | const world_distance | static | Speed threshold for normal object interpolation |
| projectile_speed_limit | const world_distance | static | Speed threshold for projectiles (higher, faster rejection) |
| previous_tick_objects, current_tick_objects | std::vector<TickObjectData> | static | Object snapshots from prior and current 30 FPS tick |
| previous_tick_polygons, current_tick_polygons | std::vector<TickPolygonData> | static | Polygon state snapshots |
| previous_tick_sides, current_tick_sides | std::vector<TickSideData> | static | Side texture coordinate snapshots |
| previous_tick_lines, current_tick_lines | std::vector<TickLineData> | static | Line height snapshots |
| previous_tick_ephemera, current_tick_ephemera | std::vector<TickObjectData> | static | Ephemera (particle effects) snapshots |
| current_tick_polygon_ephemera | std::vector<int16_t> | static | Ephemera-per-polygon list snapshot |
| previous_tick_world_view, current_tick_world_view | TickWorldView | static | Camera state snapshots |
| previous_tick_weapon_display, current_tick_weapon_display | std::vector<weapon_display_information> | static | Weapon HUD display snapshots |
| contrail_tracking | std::vector<ContrailInfo> | static | Projectile positions for contrail effects |

## Key Functions / Methods

### init_interpolated_world
- **Signature:** `void init_interpolated_world()`
- **Purpose:** Initialize all snapshot vectors and copy initial game world state into both previous and current buffers
- **Inputs:** None (reads global objects, polygons, sides, lines, ephemera arrays)
- **Outputs/Return:** None
- **Side effects:** Allocates/resizes vectors; copies world state; sets `world_is_interpolated = false`
- **Calls:** `get_fps_target()`, `get_dynamic_limit()`, `get_ephemera_data()`, `get_line_data()`, `get_weapon_display_information()`
- **Notes:** Aborts early if FPS target is 30; clears contrail tracking

### enter_interpolated_world
- **Signature:** `void enter_interpolated_world()`
- **Purpose:** Capture world state at 30 FPS tick boundary; prepare for interpolation of next frame sequence
- **Inputs:** None (reads global game state)
- **Outputs/Return:** None
- **Side effects:** Records machine tick; swaps previousΓåÉcurrent; updates all tick snapshots; sets `world_is_interpolated = true`
- **Calls:** `machine_tick_count()`, `update_world_view_camera()`, `get_weapon_display_information()`, `SLOT_IS_USED()`, `MARK_SLOT_AS_USED()`
- **Notes:** Must be called every game tick; handles dynamic side allocation from Lua scripts

### exit_interpolated_world
- **Signature:** `void exit_interpolated_world()`
- **Purpose:** Write interpolated/current-tick state back to live game world (cleanup)
- **Inputs:** None (reads current_tick_* snapshots)
- **Outputs/Return:** None
- **Side effects:** Bulk-copy current_tick_* back to game world arrays; sets `world_is_interpolated = false`
- **Calls:** None (direct array assignments)
- **Notes:** Restores world to clean state; inverse of init/enter

### update_interpolated_world
- **Signature:** `void update_interpolated_world(float heartbeat_fraction)`
- **Purpose:** Interpolate all dynamic world elements (polygons, lines, objects, ephemera) for current frame
- **Inputs:** `heartbeat_fraction` ΓÇô interpolation parameter [0, 1] where 0=previous tick, 1=next tick
- **Outputs/Return:** None
- **Side effects:** Modifies live polygon heights, side textures, line heights, object/ephemera positions
- **Calls:** `lerp()`, `should_interpolate()`, `get_object_speed_limit()`, `TEST_RENDER_FLAG()`, `find_new_object_polygon()`, `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()`, `remove_ephemera_from_polygon()`, `add_ephemera_to_polygon()`
- **Notes:** Skips invisible geometry; large function with separate loops for polygons, lines, objects, ephemera; handles polygon boundary crossings

### interpolate_world_view
- **Signature:** `void interpolate_world_view(float heartbeat_fraction)`
- **Purpose:** Interpolate camera position, angles, and depth intensity
- **Inputs:** `heartbeat_fraction` [0, 1]
- **Outputs/Return:** None
- **Side effects:** Updates `world_view->yaw/pitch/virtual_yaw/virtual_pitch/origin/maximum_depth_intensity/origin_polygon_index`
- **Calls:** `lerp_angle()`, `lerp_fixed_angle()`, `lerp()`, `find_new_object_polygon()`
- **Notes:** Skips if camera polygon undefined or camera speed exceeds limit; handles camera polygon crossings

### get_heartbeat_fraction
- **Signature:** `float get_heartbeat_fraction()`
- **Purpose:** Calculate interpolation parameter (0ΓÇô1) based on elapsed machine ticks since tick start
- **Inputs:** None (reads FPS target, movie state, replay speed)
- **Outputs/Return:** Float, nominally [0, 1] (clamped by callers)
- **Side effects:** None
- **Calls:** `Movie::instance()->IsRecording()`, `get_fps_target()`, `machine_tick_count()`, `game_is_being_replayed()`, `get_replay_speed()`
- **Notes:** Returns 1.0 for 30 FPS mode; special handling for movie export and slow-motion replay; quantizes to integer frame boundaries for fixed FPS targets

### track_contrail_interpolation
- **Signature:** `void track_contrail_interpolation(int16_t projectile_index, int16_t effect_index)`
- **Purpose:** Record projectile position at current tick for contrail/particle effect interpolation
- **Inputs:** `projectile_index` (object slot), `effect_index` (ephemera/contrail slot)
- **Outputs/Return:** None
- **Side effects:** Updates `contrail_tracking[effect_index]` with projectile polygon and location
- **Calls:** `SLOT_IS_USED()`
- **Notes:** Stores previous-tick position so contrails don't move with parent projectile

### interpolate_weapon_display_information
- **Signature:** `static void interpolate_weapon_display_information(short index, weapon_display_information* data)`
- **Purpose:** Interpolate weapon display positions (ammo counter, shell casings)
- **Inputs:** `index` (display entry), `data` (reference to interpolate)
- **Outputs/Return:** None (modifies `*data` in place)
- **Side effects:** Updates vertical/horizontal positions in `*data`
- **Calls:** `get_heartbeat_fraction()`, `lerp()`
- **Notes:** Searches circular buffer for matching shell casing data; skips if data unchanged

### Utility interpolation functions
- **`lerp(int16_t a, int16_t b, float t)` / `lerp(_fixed a, _fixed b, float t)`**: Linear interpolation with rounding
- **`lerp_angle(angle a, angle b, float t)`**: Angle interpolation with wraparound handling (chooses shortest path)
- **`lerp_fixed_angle(fixed_angle a, fixed_angle b, float t)`**: High-precision angle interpolation
- **`should_interpolate(...)`**: Rejects interpolation if object/view moved too fast (checks 2D distance)
- **`get_object_speed_limit(const TickObjectData*)`**: Returns speed threshold (higher for projectiles, lower for other objects)

## Control Flow Notes

**Lifecycle:**
1. **Startup** ΓåÆ `init_interpolated_world()` (allocate buffers, copy initial state)
2. **Each 30 FPS tick** ΓåÆ `enter_interpolated_world()` (snapshot current world state, set start time)
3. **Each render frame** ΓåÆ `get_heartbeat_fraction()` (calc progress) ΓåÆ `update_interpolated_world(fraction)` + `interpolate_world_view(fraction)` (blend state)
4. **Shutdown** ΓåÆ `exit_interpolated_world()` (restore original world state)

**Special cases:**
- If FPS target = 30: interpolation is disabled (heartbeat always 1.0, world_is_interpolated remains false)
- During movie export: heartbeat calculated from export phase instead of machine ticks
- During replay at slow speed: heartbeat scaled by replay speed multiplier
- Fast-moving projectiles: rejected from interpolation (use current tick position only)
- Polygon crossings: objects/camera may change polygon during frame; uses `find_new_object_polygon()` to relink

## External Dependencies

- **Standard library:** `<cmath>` (std::round), `<cstdint>`, `<vector>`
- **Game world data:** `objects`, `map_polygons`, `map_sides`, `map_lines` (vectors defined in map.h); `polygon_ephemera` (extern); `world_view` (extern)
- **Limits:** `get_dynamic_limit(_dynamic_limit_ephemera)` from dynamic_limits.h
- **Ephemera:** `get_ephemera_data()`, `remove_ephemera_from_polygon()`, `add_ephemera_to_polygon()` from ephemera.h
- **Map queries:** `get_line_data()`, `find_new_object_polygon()`, `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()` from map.h
- **Rendering:** `update_world_view_camera()` (render.h/cpp), `TEST_RENDER_FLAG()` (render.h)
- **Preferences:** `get_fps_target()` from preferences.h
- **Timing:** `machine_tick_count()` (defined elsewhere), `TICKS_PER_SECOND`, `MACHINE_TICKS_PER_SECOND` (map.h constants)
- **Movie:** `Movie::instance()->IsRecording()` from Movie.h
- **Replay:** `game_is_being_replayed()`, `get_replay_speed()` (defined elsewhere)
- **Weapons:** `get_weapon_display_information()`, `weapon_display_information` type from weapons.h
