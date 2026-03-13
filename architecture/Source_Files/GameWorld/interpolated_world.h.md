# Source_Files/GameWorld/interpolated_world.h

## File Purpose
Declares the public interface for a high-frequency world interpolation system that enables smooth frame rates above 30 FPS by interpolating game state and camera view between fixed update ticks. Part of the Aleph One game engine.

## Core Responsibilities
- Lifecycle management (init, enter, exit) of interpolated world context
- Per-frame state interpolation driven by heartbeat fraction
- Camera/view interpolation for smooth visual movement
- Contrail effect tracking during interpolation
- Weapon display data retrieval for rendering

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| weapon_display_information | struct | Forward declaration; holds data needed to render weapon HUD |

## Global / File-Static State
None.

## Key Functions / Methods

### init_interpolated_world
- Signature: `void init_interpolated_world()`
- Purpose: Initialize or reset the interpolated world system at startup or level load
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes interpolation state and buffers
- Calls: Not visible from header

### enter_interpolated_world / exit_interpolated_world
- Signature: `void enter_interpolated_world(); void exit_interpolated_world()`
- Purpose: Mark frame boundaries for interpolation context
- Side effects: Sets/clears interpolation state for the current frame
- Notes: Likely called at start/end of each frame loop iteration

### update_interpolated_world
- Signature: `void update_interpolated_world(float heartbeat_fraction)`
- Purpose: Update interpolated object positions and states
- Inputs: `heartbeat_fraction` ΓÇô interpolation progress [0.0ΓÇô1.0] between fixed ticks
- Side effects: Advances internal interpolation state
- Notes: Called once per rendered frame; fraction drives smooth motion

### interpolate_world_view
- Signature: `void interpolate_world_view(float heartbeat_fraction)`
- Purpose: Interpolate camera position and orientation for smooth rendering
- Inputs: `heartbeat_fraction` ΓÇô same interpolation parameter
- Side effects: Updates view matrix or camera state

### track_contrail_interpolation / get_interpolated_weapon_display_information
- Signatures: `void track_contrail_interpolation(int16_t projectile_index, int16_t effect_index); bool get_interpolated_weapon_display_information(short* count, weapon_display_information* data)`
- Purpose: Track visual effects (contrails) and retrieve weapon HUD data
- Notes: Weapon retrieval returns bool (success/failure); count and data filled on success

## Control Flow Notes
Fits into the per-frame rendering pipeline:  
1. `init_interpolated_world()` ΓÇô startup phase  
2. Each frame: `enter_interpolated_world()` ΓåÆ `update_interpolated_world(fraction)` ΓåÆ `interpolate_world_view(fraction)` ΓåÆ render ΓåÆ `exit_interpolated_world()`  
3. Contrails and weapon display queried during render setup

## External Dependencies
- `<cstdint>` ΓÇô `int16_t` type
- `weapon_display_information` struct ΓÇô defined elsewhere (likely in HUD/weapon rendering code)
