# Source_Files/GameWorld/platforms.cpp

## File Purpose
Implements dynamic platform and door mechanics for the game world. Manages platform creation, movement, collision detection, state changes, adjacent platform cascades, and interaction with world geometry including media. Supports both player and monster interaction with platforms and provides serialization/XML configuration.

## Core Responsibilities
- Create and initialize platforms in the world (`new_platform`)
- Update platform movement and state each frame (`update_platforms`)
- Activate/deactivate platforms and cascade state to adjacent platforms (`set_platform_state`, `set_adjacent_platform_states`)
- Calculate and maintain platform height extrema for movement bounds (`calculate_platform_extrema`)
- Detect and handle collisions/obstructions during platform movement
- Update geometry (endpoint heights, line solidity/transparency, texture coordinates) when platform heights change
- Manage platform-media interactions (water/lava submersion tracking)
- Play contextual sounds (starting, stopping, blocked, uncontrollable)
- Evaluate AI accessibility to/from platforms (`monster_can_enter_platform`, `monster_can_leave_platform`)
- Handle player activation and control of platforms (`player_touch_platform_state`)
- Serialize/deserialize platform data for save/load
- Parse and apply XML configuration for platform type definitions (`parse_mml_platforms`)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `platform_data` | struct | Runtime state: position, heights, movement direction, flags, endpoint adjacency info |
| `static_platform_data` | struct | Static configuration: type, speed, delay, height bounds, polygon, tag, flags |
| `platform_definition` | struct | Type definition: sound indexes, damage, key item, from `platform_definitions.h` |
| `endpoint_owner_data` | struct | Cached polygon/line ownership for an endpoint, used for height recalculation |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `platforms` | `platform_data[]` (via PlatformList) | global | All active platforms in current level |
| `dynamic_world->platform_count` | int16 | implicit | Count of platforms in current level |
| `original_platform_definitions` | `platform_definition*` | static | Backup for MML parsing reset |

## Key Functions / Methods

### new_platform
- Signature: `short new_platform(struct static_platform_data *data, short polygon_index)`
- Purpose: Create and initialize a platform from static configuration data
- Inputs: `data` (static config), `polygon_index` (polygon to attach platform to)
- Outputs/Return: Platform index (or NONE if at capacity)
- Side effects: Allocates platform slot, marks polygon as platform type, calculates extrema, initializes geometry
- Calls: `calculate_platform_extrema`, `calculate_endpoint_polygon_owners`, `calculate_endpoint_line_owners`, `adjust_platform_endpoint_and_line_heights`, `adjust_platform_for_media`
- Notes: Initializes to fully contracted/extended state based on flags; sets up endpoint owner caches for fast height recalculation

### update_platforms
- Signature: `void update_platforms(void)`
- Purpose: Advance all active platforms one tick; handle movement, collision, state transitions
- Inputs: None (reads `platforms[]`, `dynamic_world->platform_count`)
- Outputs/Return: None
- Side effects: Updates platform heights, checks/handles collisions, changes active state, plays sounds, updates media submersion flags
- Calls: `get_platform_definition`, `change_polygon_height`, `adjust_platform_sides`, `adjust_platform_endpoint_and_line_heights`, `adjust_platform_for_media`, `take_out_the_garbage`, `set_adjacent_platform_states`, `set_platform_state`, `play_platform_sound`, `guess_side_lightsource_indexes`
- Notes: Core game loop function; handles direction reversal on obstruction, auto-deactivation at levels, cascading adjacent activation

### set_platform_state
- Signature: `bool set_platform_state(short platform_index, bool state, short parent_platform_index)`
- Purpose: Activate or deactivate a platform, with cascading side-effects
- Inputs: `platform_index`, `state` (true=activate, false=deactivate), `parent_platform_index` (NONE if not triggered by another platform)
- Outputs/Return: New state (bool)
- Side effects: Updates active/moving flags, sets delay/parent, may activate/deactivate adjacent platforms, may activate/deactivate lights, plays sound, calls Lua hook
- Calls: `set_adjacent_platform_states`, `set_light_status`, `assume_correct_switch_position`, `L_Call_Platform_Activated`, `play_platform_sound`
- Notes: Per-tick limit: once platform changes state, no further state changes that tick; respects _platform_cannot_be_externally_deactivated

### set_adjacent_platform_states
- Signature: `static void set_adjacent_platform_states(short platform_index, bool state)`
- Purpose: Recursively activate/deactivate adjacent platforms if cascading flags are set
- Inputs: Platform index, desired state
- Outputs/Return: None
- Side effects: Calls `set_platform_state` on adjacent platforms
- Calls: `polygon_index_to_platform_index`, `set_platform_state`
- Notes: Prevents cycles by checking _platform_does_not_activate_parent flag

### calculate_platform_extrema
- Signature: `static void calculate_platform_extrema(short platform_index, world_distance lowest_level, world_distance highest_level)`
- Purpose: Compute min/max floor and ceiling heights for platform movement bounds
- Inputs: Platform index, optional explicit bounds (NONE = calculate from adjacent polygons)
- Outputs/Return: None (writes to platform.minimum/maximum_{floor,ceiling}_height)
- Side effects: Updates platform height bounds
- Calls: None (reads geometry only)
- Notes: Complex logic: floor platforms extend up to highest adjacent floor (or polygon floor if _platform_uses_native_polygon_heights); ceiling platforms extend down to lowest adjacent ceiling; split platforms meet in middle

### adjust_platform_endpoint_and_line_heights
- Signature: `void adjust_platform_endpoint_and_line_heights(short platform_index)`
- Purpose: Recalculate all endpoint and line solidity/transparency after platform height change
- Inputs: Platform index
- Outputs/Return: None
- Side effects: Updates line.highest_adjacent_floor, line.lowest_adjacent_ceiling, endpoint.highest_adjacent_floor_height, etc.; updates line/endpoint solidity and transparency flags
- Calls: `get_map_indexes` (via endpoint_owners cache)
- Notes: Essential for correct collision geometry and visual clipping after movement

### adjust_platform_sides
- Signature: `void adjust_platform_sides(platform_data* platform, world_distance old_ceiling_height, world_distance new_ceiling_height)`
- Purpose: Update texture coordinates on platform sides when ceiling platform moves
- Inputs: Platform, old and new ceiling heights
- Outputs/Return: None
- Side effects: Adjusts primary_texture.y0 on sides (vertical texture scroll to simulate sliding)
- Calls: None
- Notes: Only applies to ceiling platforms (_high_side, _split_side, _full_side); prevents texture stretching/swimming as platform moves

### monster_can_enter_platform
- Signature: `short monster_can_enter_platform(short platform_index, short source_polygon_index, world_distance height, world_distance minimum_ledge_delta, world_distance maximum_ledge_delta)`
- Purpose: Determine if a monster can physically move from source polygon onto platform
- Inputs: Platform, source polygon, monster height, acceptable ledge height range
- Outputs/Return: `_platform_is_accessable`, `_platform_will_be_accessable`, or `_platform_will_never_be_accessable`
- Side effects: None
- Calls: None
- Notes: Special handling for doors (uses contracted/extended bounds) vs. platforms; evaluates floor delta, ceiling clearance

### monster_can_leave_platform
- Signature: `short monster_can_leave_platform(short platform_index, short destination_polygon_index, world_distance height, world_distance minimum_ledge_delta, world_distance maximum_ledge_delta)`
- Purpose: Determine if a monster can move from platform to destination polygon
- Inputs: Platform, destination polygon, monster height, acceptable ledge height range
- Outputs/Return: `_exit_is_accessable`, `_exit_will_be_accessable`, or `_exit_will_never_be_accessable`
- Side effects: None
- Calls: None
- Notes: Mirror of `monster_can_enter_platform`

### player_touch_platform_state
- Signature: `void player_touch_platform_state(short player_index, short platform_index)`
- Purpose: Handle player pressing action key on a platform
- Inputs: Player index, platform index
- Outputs/Return: None
- Side effects: May activate/deactivate platform, reverse direction, or play uncontrollable sound; may consume player item (key)
- Calls: `set_platform_state`, `try_and_subtract_player_item`, `play_platform_sound`
- Notes: Respects _platform_is_player_controllable flag; may require key item for activation

### play_platform_sound
- Signature: `static void play_platform_sound(short platform_index, short type)`
- Purpose: Play appropriate sound for platform state change
- Inputs: Platform index, sound type enum (_starting_sound, _stopping_sound, _obstructed_sound, _uncontrollable_sound)
- Outputs/Return: None
- Side effects: Plays polygon sound, updates ambient sound sources
- Calls: `play_polygon_sound`, `SoundManager::instance()->CauseAmbientSoundSourceUpdate()`
- Notes: Selects sound from platform_definition based on direction (extending vs. contracting)

### adjust_platform_for_media
- Signature: `void adjust_platform_for_media(short platform_index, bool initialize)`
- Purpose: Update media submersion flags and play entry/exit sounds
- Inputs: Platform index, whether this is initialization
- Outputs/Return: None
- Side effects: Updates _platform_floor_below_media, _platform_ceiling_below_media flags; plays splash sounds
- Calls: `play_polygon_sound`
- Notes: Only relevant if polygon has media; tracks transitions for sound generation

### unpack_platform_data / pack_platform_data
- Signature: `uint8 *unpack_platform_data(uint8 *Stream, platform_data* Objects, size_t Count)` / `pack`
- Purpose: Serialize/deserialize runtime platform state to/from byte stream (for save/load)
- Inputs: Byte stream, object array, count
- Outputs/Return: Pointer to next byte in stream
- Side effects: None
- Calls: `StreamToValue` / `ValueToStream` macros
- Notes: Packing format matches SIZEOF_platform_data (140 bytes per platform)

### parse_mml_platforms
- Signature: `void parse_mml_platforms(const InfoTree& root)`
- Purpose: Parse XML configuration to override platform type definitions
- Inputs: XML tree root
- Outputs/Return: None
- Side effects: Backs up original definitions, updates platform_definitions array
- Calls: `reset_mml_platforms` (on first call to backup), InfoTree methods
- Notes: Supports indexing into platform types by index; parses sound and damage overrides

## Control Flow Notes
**Initialization phase** (`new_platform`): Platforms are created, geometry is calculated, endpoint owner caches are built, initial state is set (active/inactive, extended/contracted).

**Per-frame update** (`update_platforms`): Called once per game tick. Each active platform checks if it should move, applies delta height with collision checking, updates geometry if unobstructed, and handles state transitions (reaching end of motion, deactivation). Geometry updates propagate to endpoints and lines.

**State change** (`set_platform_state` / `player_touch_platform_state`): Can occur at any time, subject to per-tick guard. May trigger cascading adjacent platform changes.

Not explicitly inferable: Exact tick timing, initial load/parse of platform_definitions.

## External Dependencies
- **world.h**: World coordinate types (`world_distance`, `world_point2d`, `world_point3d`), geometry utilities
- **map.h**: Map structures (`polygon_data`, `line_data`, `endpoint_data`, `side_data`), polygon/line/endpoint accessors, `dynamic_world`, `change_polygon_height`
- **platforms.h**: Type definitions (`platform_data`, `static_platform_data`), flag macros, function prototypes
- **lightsource.h**: `set_light_status` for platform-controlled lights
- **SoundManager.h**: Sound playback (`play_polygon_sound`, `SoundManager::instance()->CauseAmbientSoundSourceUpdate`)
- **player.h**: `try_and_subtract_player_item` for key consumption, player item access
- **media.h**: `get_media_data` for media height queries
- **lua_script.h**: `L_Call_Platform_Activated` (Pfhortran scripting hook)
- **InfoTree.h**: XML parsing for MML configuration
- **items.h**, **Packing.h**: Damage definition parsing (included but minimal use visible)
- **editor.h**: `MARATHON_ONE_DATA_VERSION` constant for version-specific unpacking
