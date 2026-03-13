# Source_Files/GameWorld/platforms.h

## File Purpose

Defines the platform system for the Marathon/Aleph One game engine. Platforms are dynamic geometry elements (doors, rising platforms, etc.) that can move, activate based on events, and respond to player/monster interaction.

## Core Responsibilities

- Define 9 platform type constants and speed/delay enumerations
- Manage 32 static (design-time) and 16 dynamic (runtime) state flags via bitwise macros
- Define `static_platform_data` (32-byte design template) and `platform_data` (140-byte runtime instance) structures
- Declare platform lifecycle functions: creation, activation, state changes, media interaction
- Declare platform accessibility queries for pathfinding (can monsters traverse?)
- Declare serialization and MML-based data loading
- Maintain the global `PlatformList` vector as the authority on all map platforms

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| static_platform_data | struct | 32-byte immutable template: type, speed, delay, heights, flags, polygon, tag |
| platform_data | struct | 140-byte runtime instance: extends static data + dynamic state, endpoint owners, parent platform, heights |
| endpoint_owner_data | struct | Tracks polygon and line indices owned by a platform endpoint |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| PlatformList | `vector<platform_data>` | extern | All active platforms in current map; accessed via `platforms` macro for C-style compatibility |

## Key Functions / Methods

### new_platform
- Signature: `short new_platform(static_platform_data *data, short polygon_index)`
- Purpose: Instantiate a new platform from static template
- Inputs: design template, owning polygon index
- Outputs: platform index
- Side effects: Allocates and initializes platform_data in PlatformList

### get_defaults_for_platform_type
- Signature: `static_platform_data *get_defaults_for_platform_type(short type)`
- Purpose: Retrieve default static_platform_data for a type (door, platform, etc.)
- Inputs: platform type enum (0ΓÇô8)
- Outputs: pointer to const/default static_platform_data

### update_platforms
- Signature: `void update_platforms(void)`
- Purpose: Main per-tick update loop; drives all active platform movement and state transitions
- Calls: Likely called by main game loop once per tick

### try_and_change_platform_state
- Signature: `bool try_and_change_platform_state(short platform_index, bool state)`
- Purpose: Attempt to activate (`true`) or deactivate (`false`) a single platform
- Inputs: platform index, desired state
- Outputs: `true` if state changed, `false` if blocked or invalid
- Notes: Respects flags like `_platform_activates_only_once`, `_platform_cannot_be_externally_deactivated`; may trigger adjacent platform chains

### try_and_change_tagged_platform_states
- Signature: `bool try_and_change_tagged_platform_states(short tag, bool state)`
- Purpose: Change state of all platforms with matching tag (used by control panels, triggers)
- Inputs: tag value, desired state
- Outputs: `true` if any platform changed
- Calls: Likely calls `try_and_change_platform_state` for each matching platform

### monster_can_enter_platform / monster_can_leave_platform
- Signatures: `short monster_can_enter_platform(short platform_index, short source_polygon_index, world_distance height, world_distance minimum_ledge_delta, world_distance maximum_ledge_delta)` and similar for exit
- Purpose: Pathfinding query; determine if a monster can traverse platform at given height constraints
- Inputs: platform, source/destination polygon, entity height, ledge height bounds
- Outputs: enum indicating accessibility level (`_platform_will_never_be_accessible`, `_platform_will_be_accessible`, `_platform_might_be_accessible`, `_platform_is_accessible`, and exit variants)
- Notes: Used by monster AI for navigation decisions

### adjust_platform_for_media
- Signature: `void adjust_platform_for_media(short platform_index, bool initialize)`
- Purpose: Recalculate platform position/accessibility when liquid media enters/leaves
- Inputs: platform index, `true` if initializing level
- Side effects: Updates floor/ceiling heights, sets `_platform_floor_below_media` / `_platform_ceiling_below_media` flags

### player_touch_platform_state
- Signature: `void player_touch_platform_state(short player_index, short platform_index)`
- Purpose: Handle player pressing action key on a platform control panel
- Inputs: player and platform indices
- Notes: Triggered by player interaction with player-controllable platforms

### get_platform_data
- Signature: `platform_data *get_platform_data(short platform_index)`
- Purpose: Safe accessor for platform data by index
- Outputs: pointer to platform_data or assertion/null on out-of-bounds

### Serialization Functions
`unpack_static_platform_data`, `pack_static_platform_data`, `unpack_platform_data`, `pack_platform_data`
- Purpose: Convert platform structures to/from byte streams for save files and network transmission
- Inputs/Outputs: pointer to byte stream, array of objects, count
- Notes: Return updated stream pointer for chaining

### parse_mml_platforms / reset_mml_platforms
- Signatures: `void parse_mml_platforms(const InfoTree& root)`, `void reset_mml_platforms(void)`
- Purpose: Load platform configuration from MML (Marathon Markup Language) XML; reset to defaults
- Notes: Decouples platform behavior from hardcoded engine values

**Notes on trivial/helper macros:** Extensive `PLATFORM_IS_*` (read) and `SET_PLATFORM_*` (write) macros wrap `TEST_FLAG16`/`SET_FLAG16` and `TEST_FLAG32`/`SET_FLAG32` for safe flag access. ~50 macros defined; not enumerated individually.

## Control Flow Notes

**Init/Load:**
- `new_platform()` creates instances from static_platform_data templates
- `adjust_platform_for_media(..., true)` initializes media-aware platforms

**Per-Frame Update:**
- `update_platforms()` drives all active platforms: increment heights toward target, handle collisions, trigger state changes
- Platforms transition through states: inactive ΓåÆ active (with optional delay) ΓåÆ extending ΓåÆ fully extended ΓåÆ contracting ΓåÆ fully contracted ΓåÆ return to inactive

**Activation Triggers:**
- Player/monster entry ΓåÆ `platform_was_entered()`
- Player action key ΓåÆ `player_touch_platform_state()` (if `_platform_is_player_controllable`)
- Monster path validation ΓåÆ `monster_can_enter/leave_platform()` queries
- Control panel / trigger ΓåÆ `try_and_change_platform_state()` or `try_and_change_tagged_platform_states()`
- Adjacent platform chains (if `_platform_activates_adjacent_platforms_when_*` flags set)

**Media Interaction:**
- Liquid entry/exit ΓåÆ `adjust_platform_for_media()` updates heights and accessibility flags

**Shutdown/Serialization:**
- `pack_platform_data()` before save or network send
- `unpack_platform_data()` after load or network receive

## External Dependencies

- **map.h**: polygon_data, line_data, endpoint_data, world_distance, world_point3d, side_data, MAXIMUM_VERTICES_PER_POLYGON
- **Macros (via map.h or csmacros.h):** TEST_FLAG16, SET_FLAG16, TEST_FLAG32, SET_FLAG32
- **Constants:** WORLD_ONE, TICKS_PER_SECOND
- **MML parsing:** InfoTree (likely from XML/data module)
- **Bungie/Aleph One licensing:** GNU GPL v3
