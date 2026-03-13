# Source_Files/GameWorld/weapons.h

## File Purpose
Header for the weapon management system in Aleph One (Marathon engine). Defines structures, enumerations, and function prototypes for managing player weapons, ammunition, firing state, display rendering, and serialization.

## Core Responsibilities
- Define weapon types and pseudo-weapons (fist, pistol, plasma pistol, assault rifle, SMG, shotgun, flamethrower, alien shotgun, missile launcher, etc.)
- Track per-player weapon state: current weapon, desired weapon, ammo counts, trigger state, animation frame
- Manage weapon display information for rendering in game window (collection, shape index, positioning, transfer mode)
- Handle shell casing rendering and animation
- Initialize weapons for new games and new players
- Update weapons each frame based on player input (action flags)
- Serialize/deserialize weapon state and definitions (pack/unpack functions)
- Load/unload weapon resource collections
- Process ammo pickups and reloading
- Support XML/MML configuration of weapon parameters
- Query weapon state for UI and rendering systems

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `weapon_display_information` | struct | Animation frame, positioning, and rendering mode for 3D weapon model display |
| `trigger_data` | struct | State of a single trigger: animation phase, ammo, shots fired/hit, timing, sequence |
| `weapon_data` | struct | Complete weapon state: type, flags, two triggers (primary/secondary) |
| `shell_casing_data` | struct | Position, velocity, type, frame for animated shell casings |
| `player_weapon_data` | struct | All weapons for one player: current/desired weapon, array of weapon states, shell casings |
| Weapon type enum | enum | `_weapon_fist` through `_weapon_smg`, plus pseudo-weapons (doublefisted variants) |
| Weapon action enum | enum | `_shape_weapon_idle`, `_shape_weapon_charging`, `_shape_weapon_firing` |
| Trigger enum | enum | `_primary_weapon`, `_secondary_weapon` |
| Display position enum | enum | `_position_low`, `_position_center`, `_position_high` (for UI positioning) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SIZEOF_weapon_definition` | const int (134) | global | Size for serialization of weapon definitions |
| `SIZEOF_player_weapon_data` | const int (472) | global | Size for serialization of player weapon state |

## Key Functions / Methods

### initialize_weapon_manager
- **Signature:** `void initialize_weapon_manager(void)`
- **Purpose:** One-time startup initialization of the weapon system
- **Calls:** (defined elsewhere)

### initialize_player_weapons_for_new_game
- **Signature:** `void initialize_player_weapons_for_new_game(short player_index)`
- **Purpose:** Reset all weapons to initial state for a new game session
- **Inputs:** Player index
- **Calls:** (defined elsewhere)

### update_player_weapons
- **Signature:** `void update_player_weapons(short player_index, uint32 action_flags)`
- **Purpose:** Main per-frame weapon update; processes firing, reloading, animations
- **Inputs:** Player index, action flags (player input bitmask)
- **Side effects:** Modifies weapon state, ammo, animation frame
- **Calls:** (defined elsewhere)

### get_weapon_display_information
- **Signature:** `bool get_weapon_display_information(short *count, struct weapon_display_information *data)`
- **Purpose:** Iterator to retrieve weapon display frames for rendering
- **Inputs:** Pointer to count (in), pointer to display info struct (out)
- **Outputs:** Returns true while more frames available; populates data struct
- **Notes:** Must be called repeatedly until false

### process_new_item_for_reloading
- **Signature:** `void process_new_item_for_reloading(short player_index, short item_type)`
- **Purpose:** Handle ammo/weapon pickups and reloading
- **Inputs:** Player index, item type

### player_hit_target
- **Signature:** `void player_hit_target(short player_index, short weapon_identifier)`
- **Purpose:** Feedback when a weapon projectile hits; updates hit counters
- **Inputs:** Player index, weapon identifier from projectile

### Serialization functions
- **unpack_player_weapon_data**, **pack_player_weapon_data**: Serialize complete weapon state
- **unpack_weapon_definition**, **pack_weapon_definition**: Serialize weapon parameters
- **unpack_m1_weapon_definition**: Legacy Marathon 1 format support
- **All take:** `uint8 *Stream, size_t Count`; return `uint8*` (stream pointer for chaining)

### Query/State functions
- **get_player_weapon_mode_and_type** ΓåÆ `(shape_type, shape_mode)` for rendering
- **get_player_desired_weapon** ΓåÆ `short` for UI display
- **get_player_weapon_ammo_count/maximum/type/drawn** ΓåÆ Ammo queries for UI

### Resource and config
- **mark_weapon_collections** ΓåÆ Load/unload weapon graphics collections
- **discharge_charged_weapons** ΓåÆ Cleanup on player death
- **check_player_weapons_for_environment_change** ΓåÆ Validate weapons on level entry
- **parse_mml_weapons**, **reset_mml_weapons** ΓåÆ XML/MML configuration
- **init_weapon_definitions** ΓåÆ Load weapon parameter definitions
- **get_number_of_weapon_types** ΓåÆ `size_t` query

## Control Flow Notes
- **Init phase:** `initialize_weapon_manager()` at startup, `init_weapon_definitions()` loads XML/MML config
- **Per-player setup:** `initialize_player_weapons_for_new_game()` or `initialize_player_weapons()`
- **Frame loop:** `update_player_weapons()` called each frame with input flags
- **Rendering:** `get_weapon_display_information()` called iteratively to fetch animation frames
- **Events:** `process_new_item_for_reloading()` on pickup, `player_hit_target()` on projectile hit, `discharge_charged_weapons()` on death
- **Level change:** `check_player_weapons_for_environment_change()`

## External Dependencies
- **cstypes.h** (bundled): Provides `_fixed`, `uint8`, `uint16`, `uint32`, `int16`, `int32`
- **InfoTree** class: (defined elsewhere) Used for XML/MML parsing
- **Resource system:** Weapon collections managed by external code (see `mark_weapon_collections`)
- **Projectile system:** Receives feedback via `player_hit_target()` (defined elsewhere)
- **Comment reference:** lua_script.cpp accesses shell casing constants
