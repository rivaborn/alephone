# Source_Files/RenderOther/ChaseCam.h

## File Purpose
This header defines the public interface for a third-person chase camera system (similar to Halo). It enables a follow-behind-the-player camera with configurable physics, optional side-switching for wall avoidance, and optional through-wall rendering. The implementation is distributed across multiple compilation units.

## Core Responsibilities
- Define chase camera state and configuration parameters (`ChaseCamData` struct)
- Provide game initialization and level transition hooks (`ChaseCam_Initialize`, `ChaseCam_Reset`)
- Manage per-frame physics updates (`ChaseCam_Update`)
- Control camera active state and side-switching behavior
- Query camera feasibility and retrieve final view position/orientation for rendering
- Expose configuration dialog and preference loading interfaces

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| ChaseCamData | struct | Stores camera parameters: `Behind` (distance), `Upward` (height), `Rightward` (side offset), `Flags` (behavior control), `Damping` (motion damping), `Spring` (spring constant), `Opacity` (blend factor) |
| Chase-cam flags | enum | Bit flags: `_ChaseCam_OnWhenEntering` (auto-activate on level load), `_ChaseCam_NeverActive` (disable), `_ChaseCam_ThroughWalls` (ignore geometry) |

## Global / File-Static State
None.

## Key Functions / Methods

### Configure_ChaseCam
- Signature: `bool Configure_ChaseCam(ChaseCamData &Data)`
- Purpose: Present configuration dialog to user
- Inputs: Reference to `ChaseCamData` struct to modify
- Outputs/Return: `true` if OK, `false` if canceled; struct unchanged on cancel
- Side effects: UI interaction
- Calls: None visible (implemented in PlayerDialogs.c)
- Notes: User-facing configuration; changes not persisted by this call

### GetChaseCamData
- Signature: `ChaseCamData& GetChaseCamData()`
- Purpose: Retrieve active chase camera configuration from preferences
- Inputs: None
- Outputs/Return: Reference to active `ChaseCamData`
- Side effects: None
- Calls: None visible (implemented in preferences.c)
- Notes: Preference accessor; returns live configuration

### ChaseCam_CanExist
- Signature: `bool ChaseCam_CanExist()`
- Purpose: Check if chase camera can possibly activate (used to avoid loading player sprites if unnecessary)
- Inputs: None
- Outputs/Return: `true` if camera activation is feasible
- Side effects: None
- Calls: None visible
- Notes: Optimization hook

### ChaseCam_IsActive / ChaseCam_SetActive
- Signature: `bool ChaseCam_IsActive()` / `bool ChaseCam_SetActive(bool NewState)`
- Purpose: Query or change active state
- Inputs: (`SetActive`) `NewState` - desired state
- Outputs/Return: Current or resulting active state
- Side effects: (`SetActive`) Changes global camera state if conditions permit
- Calls: None visible

### ChaseCam_Initialize
- Signature: `bool ChaseCam_Initialize()`
- Purpose: Initialize chase camera at game start
- Inputs: None
- Outputs/Return: `true` if successful
- Side effects: Allocates/initializes internal state
- Calls: None visible

### ChaseCam_Reset
- Signature: `bool ChaseCam_Reset()`
- Purpose: Reset camera when level changes, player revives, or teleports
- Inputs: None
- Outputs/Return: `true` if successful
- Side effects: Resets position/velocity without deallocating

### ChaseCam_Update
- Signature: `bool ChaseCam_Update()`
- Purpose: Advance camera physics by one game tick
- Inputs: None
- Outputs/Return: `true` if successful
- Side effects: Updates position/velocity using spring and damping constants
- Calls: None visible
- Notes: Must be called every frame for correct physics behavior

### ChaseCam_SwitchSides
- Signature: `bool ChaseCam_SwitchSides()`
- Purpose: Toggle camera horizontal offset (left Γåö right) to avoid walls
- Inputs: None
- Outputs/Return: `true` if successful
- Side effects: Modifies camera offset

### ChaseCam_GetPosition
- Signature: `bool ChaseCam_GetPosition(world_point3d &position, short &polygon_index, angle &yaw, angle &pitch)`
- Purpose: Retrieve final camera position and orientation for rendering
- Inputs: References to output variables
- Outputs/Return: `true` if active; fills position (3D world space), polygon location, yaw, and pitch
- Side effects: None (read-only if inactive; outputs unchanged when camera inactive)
- Calls: None visible

## Control Flow Notes
Typical frame loop:
1. `ChaseCam_Initialize()` at game start
2. `ChaseCam_Reset()` on level load or teleport
3. `ChaseCam_Update()` once per tick (physics)
4. `ChaseCam_GetPosition()` to retrieve view for renderer
5. Optional `ChaseCam_SetActive()` / `ChaseCam_SwitchSides()` on input

## External Dependencies
- **Includes:** `world.h` (defines `world_point3d`, `angle`, `world_distance`)
- **Implemented elsewhere:**
  - `Configure_ChaseCam()` ΓåÆ PlayerDialogs.c
  - `GetChaseCamData()` ΓåÆ preferences.c
- **Inferred callers:** Game loop, player sprite loader, renderer
