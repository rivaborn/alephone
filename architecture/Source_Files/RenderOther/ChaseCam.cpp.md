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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| ChaseCamData | struct | Configuration: distance behind player, vertical offset, horizontal offset, damping/spring constants, opacity (defined in ChaseCam.h) |
| world_point3d | struct | 3D world coordinates (x, y, z); includes orientation angles |
| polygon_data | struct | Map polygon geometry with floor/ceiling heights (from map.h) |
| line_data | struct | Map boundary line between polygons (from map.h) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| _ChaseCam_IsActive | bool | static | Tracks whether chase cam is currently active |
| _ChaseCam_IsReset | bool | static | Flag indicating reset state (used to suppress inertia on first frame after reset) |
| CC_Position | world_point3d | static | Current camera world position |
| CC_Position_1 | world_point3d | static | Previous frame's camera position (for damping calculation) |
| CC_Position_2 | world_point3d | static | Two frames ago position (for spring physics) |
| CC_Polygon | short | static | Current polygon containing camera |
| CC_Yaw | angle | static | Camera horizontal rotation (synchronized with player facing) |
| CC_Pitch | angle | static | Camera vertical rotation (synchronized with player elevation) |

## Key Functions / Methods

### ChaseCam_CanExist
- **Signature:** `bool ChaseCam_CanExist()`
- **Purpose:** Check if chase cam can ever be activated (respects configuration flag to disable completely)
- **Inputs:** None (reads GetChaseCamData())
- **Outputs/Return:** Boolean; true if chase cam is not permanently disabled
- **Side effects:** None
- **Calls:** GetChaseCamData(), TEST_FLAG()
- **Notes:** Used to avoid loading player sprite graphics if chase cam will never be needed

### ChaseCam_IsActive
- **Signature:** `bool ChaseCam_IsActive()`
- **Purpose:** Query whether chase cam is currently rendering
- **Inputs:** None
- **Outputs/Return:** Boolean; true if active and allowed by network settings
- **Side effects:** None
- **Calls:** NetAllowBehindview(), ChaseCam_CanExist()
- **Notes:** Network and capability checks prevent activation in multiplayer if behind-view is disabled

### ChaseCam_SetActive
- **Signature:** `bool ChaseCam_SetActive(bool NewState)`
- **Purpose:** Activate or deactivate the chase camera
- **Inputs:** NewState: desired activation state
- **Outputs/Return:** Boolean; new active state (may differ from request if preconditions fail)
- **Side effects:** Sets _ChaseCam_IsActive flag; calls ChaseCam_Reset() and ChaseCam_Update() if transitioning on
- **Calls:** NetAllowBehindview(), ChaseCam_CanExist(), ChaseCam_Reset(), ChaseCam_Update()
- **Notes:** Performs immediate update when activated to position camera correctly

### ChaseCam_Initialize
- **Signature:** `bool ChaseCam_Initialize()`
- **Purpose:** Initialize chase cam for a new game/level
- **Inputs:** None (reads GetChaseCamData().Flags)
- **Outputs/Return:** Boolean; true if successfully initialized
- **Side effects:** Calls ChaseCam_Reset() and ChaseCam_SetActive() based on configuration
- **Calls:** ChaseCam_CanExist(), ChaseCam_Reset(), ChaseCam_SetActive(), GetChaseCamData()
- **Notes:** Honors _ChaseCam_OnWhenEntering flag to auto-activate on level entry

### ChaseCam_Reset
- **Signature:** `bool ChaseCam_Reset()`
- **Purpose:** Reset camera state (after teleport or level change) to suppress inertia momentarily
- **Inputs:** None
- **Outputs/Return:** Boolean; true if reset was applied
- **Side effects:** Sets _ChaseCam_IsReset flag; next update will reinitialize position history
- **Calls:** ChaseCam_CanExist(), ChaseCam_IsActive()
- **Notes:** Prevents jittery camera motion when player teleports or level loads

### ChaseCam_SwitchSides
- **Signature:** `bool ChaseCam_SwitchSides()`
- **Purpose:** Flip camera horizontal offset (left Γåö right)
- **Inputs:** None (modifies GetChaseCamData())
- **Outputs/Return:** Boolean; true if switch succeeded
- **Side effects:** Negates ChaseCam.Rightward value
- **Calls:** ChaseCam_CanExist(), ChaseCam_IsActive(), GetChaseCamData()
- **Notes:** Allows player to view around player from either shoulder

### ChaseCam_GetPosition
- **Signature:** `bool ChaseCam_GetPosition(world_point3d &position, short &polygon_index, angle &yaw, angle &pitch)`
- **Purpose:** Retrieve current camera position and orientation (for rendering)
- **Inputs:** None (reads global camera state)
- **Outputs/Return:** Boolean; outputs by reference: position, polygon, yaw, pitch. Returns true if active.
- **Side effects:** None (read-only, does not modify outputs if inactive)
- **Calls:** ChaseCam_CanExist(), ChaseCam_IsActive()
- **Notes:** Outputs are only valid if function returns true; renderer must not use them otherwise

### CC_PosUpdate
- **Signature:** `static int CC_PosUpdate(float Damping, float Spring, short x0, short x1, short x2)`
- **Purpose:** Apply damping and spring physics to smooth position between previous frames
- **Inputs:** Damping: damping coefficient; Spring: spring stiffness; x0, x1, x2: current, previous, two-frames-ago positions
- **Outputs/Return:** Integer; new damped/sprung position
- **Side effects:** None
- **Calls:** None
- **Notes:** Uses formula: `x = x0 + 2*damping*(x1-x0) - (damping┬▓+spring)*(x2-x0)`; result is rounded to nearest integer

### ShootForTargetPoint
- **Signature:** `static void ShootForTargetPoint(bool ThroughWalls, world_point3d& StartPosition, world_point3d& EndPosition, short& Polygon)`
- **Purpose:** Trace a ray from start to end position, clipping against world geometry and updating endpoint/polygon if collision detected
- **Inputs:** ThroughWalls: if false, camera cannot pass through solid boundaries; StartPosition: ray origin; EndPosition: target (modified in-place); Polygon: starting polygon index (modified in-place)
- **Outputs/Return:** None (modifies EndPosition and Polygon by reference)
- **Side effects:** Modifies EndPosition and Polygon to reflect final collision-aware position
- **Calls:** get_polygon_data(), find_floor_or_ceiling_intersection(), find_line_crossed_leaving_polygon(), get_line_data(), get_endpoint_data(), find_line_intersection(), find_adjacent_polygon()
- **Notes:** Handles wraparound in short integer coordinates; checks floors, ceilings, and walls in sequence; if ThroughWalls is true, tracks last valid polygon but does not clamp position

### ChaseCam_Update
- **Signature:** `bool ChaseCam_Update()`
- **Purpose:** Update camera position and orientation once per game tick
- **Inputs:** None (reads current_player, ChaseCamData, static position history)
- **Outputs/Return:** Boolean; true if active
- **Side effects:** Updates CC_Position, CC_Polygon, CC_Yaw, CC_Pitch; modifies position history (CC_Position_1, CC_Position_2) and _ChaseCam_IsReset flag
- **Calls:** ChaseCam_CanExist(), ChaseCam_IsActive(), GetChaseCamData(), translate_point3d(), translate_point2d(), CC_PosUpdate(), ShootForTargetPoint()
- **Notes:** Main per-frame logic: shifts old positions back, applies offset from player, applies damping/spring if not reset, performs collision detection, marks reset as complete

## Control Flow Notes
**Initialization Path:**
- ChaseCam_Initialize() ΓåÆ ChaseCam_Reset() ΓåÆ optionally ChaseCam_SetActive() (if _ChaseCam_OnWhenEntering is set)

**Per-Frame Path:**
- ChaseCam_Update() runs each game tick; applies physics and updates global camera state
- ChaseCam_GetPosition() called by renderer to fetch current camera pose

**State Transitions:**
- Reset triggered by ChaseCam_Reset() (called on level load, teleport, or via player action); suppresses inertia for one frame
- Side-switch via ChaseCam_SwitchSides() negates horizontal offset; independent of position update

## External Dependencies
- **map.h:** polygon_data, line_data, endpoint_data structures; functions: get_polygon_data(), find_floor_or_ceiling_intersection(), find_line_crossed_leaving_polygon(), get_line_data(), get_endpoint_data(), find_line_intersection(), find_adjacent_polygon()
- **player.h:** player_data structure; extern current_player pointer; world_point3d and angle types
- **network.h:** NetAllowBehindview() (checks network/cheat flags)
- **world.h:** world_point3d, world_point2d, angle type definitions; WORLD_ONE, QUARTER_CIRCLE, SHRT_MAX, SHRT_MIN constants
- **ChaseCam.h:** ChaseCamData structure, GetChaseCamData() function (defined elsewhere, likely preferences.c)
