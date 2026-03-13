# Source_Files/GameWorld/player.cpp

## File Purpose
Core player management system for the Marathon game engine. Handles all aspects of player lifecycle including creation, physics, damage, death/revival, equipment management, and network serialization. Supports configurable player attributes via MML (Marathon Markup Language).

## Core Responsibilities
- Player creation, initialization, and destruction across single-player and multiplayer games
- Per-frame player state updates including physics, oxygen/energy depletion, and animation
- Damage application, death mechanics, and player respawning with optional penalties
- Powerup acquisition, duration tracking, and visual effect management
- Player equipment (weapons, items) and initial inventory distribution
- Teleportation (level-based and intra-level) handling and animation
- Action queue management and hotkey sequence decoding for weapon selection
- Terminal interaction mode with time-stop in solo play
- Player data serialization/deserialization for save games and network transmission
- MML-driven configuration of player settings, powerups, initial items, and damage responses

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| player_data | struct | Main player state: position, health, inventory, physics, team, status flags |
| damage_record | struct | Tracks cumulative damage given/taken with kill counts per player |
| player_settings_definition | struct | Configurable globals: initial energy/oxygen, powerup values, visual range, oxygen rates |
| player_powerup_durations_definition | struct | Duration values (in ticks) for invisibility, invincibility, infravision, extravision |
| player_powerup_definition | struct | Maps powerup types to item IDs for MML-configurable pickup identification |
| player_shape_definitions | struct | Indices into sprite collection for legs, torsos (idle/charging/firing), death poses |
| damage_response_definition | struct | Maps damage type to: sound effect, fade effect, death sound, death animation |
| physics_variables | struct | Player movement state: position, velocity, direction, elevation, step phase |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| players | player_data* | global | Dynamic array of all active player slots (size: dynamic_world->player_count) |
| local_player, current_player | player_data* | global | Convenience pointers to locally-controlled and camera-target players |
| local_player_index, current_player_index | short | global | Array indices for above; NONE if not set |
| team_damage_given[8], team_damage_taken[8] | damage_record[] | global | Per-team cumulative damage stats |
| team_monster_damage_taken[8], team_monster_damage_given[8] | damage_record[] | global | Per-team damage from/to monsters |
| team_friendly_fire[8] | damage_record[] | global | Per-team friendly fire tracking |
| sRealActionQueues | ActionQueues* | static | Singleton queue manager for player input dequeuing |
| player_shapes | player_shape_definitions | global | Current sprite indices; restored from original on MML reset |
| player_initial_items | short[] | global | Starting inventory item type array; can be reconfigured via MML |
| damage_response_definitions | damage_response_definition[] | global | Lookup table for damage typeΓåÆresponse behavior |
| player_settings | player_settings_definition | global | Current gameplay parameters (energy, oxygen, visual range, etc.) |
| player_powerup_durations | player_powerup_durations_definition | global | Current powerup duration values |
| player_powerups | player_powerup_definition | global | Current powerup item ID mappings |
| sLocalPlayerTicksSinceTerminal | int | static | Tick counter for terminal usage (not carefully measured) |
| original_* (6 backups) | various* | static | Preserved original values for MML reset on configuration reload |

## Key Functions / Methods

### allocate_player_memory
- Signature: `void allocate_player_memory(void)`
- Purpose: Initialize player data array and action queue system at engine startup
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates `MAXIMUM_NUMBER_OF_PLAYERS` player_data slots; creates sRealActionQueues singleton
- Calls: `new` (player_data array, ActionQueues constructor)
- Notes: Called once during initialization; uses dprintf debug output in BETA builds

### new_player
- Signature: `short new_player(short team, short color, short identifier, new_player_flags flags)`
- Purpose: Create a new player slot and initialize all associated data
- Inputs: team (allegiance), color (team color index), identifier (player start data), flags (local/current status)
- Outputs/Return: new player_index in player array
- Side effects: Increments dynamic_world->player_count; calls recreate_player(); marks inventory dirty
- Calls: `get_player_data`, `recreate_player`, `give_player_initial_items`, `try_and_strip_player_items`, `mark_player_inventory_as_dirty`, `initialize_player_weapons`
- Notes: Sets teleporting_destination to NO_TELEPORTATION_DESTINATION; initializes all powerup durations to 0

### update_players
- Signature: `void update_players(ActionQueues* inActionQueuesToUse, bool inPredictive)`
- Purpose: Main per-frame update for all active players; processes input, applies physics, handles damage/death
- Inputs: inActionQueuesToUse (action queue manager to dequeue from), inPredictive (if true, runs reduced state update for prediction rollback)
- Outputs/Return: None
- Side effects: Modifies all player state (location, facing, health, oxygen, animations); may trigger death/revival; updates terminal interaction state
- Calls: `dequeueActionFlags`, `PLAYER_IS_TELEPORTING`, `player_in_terminal_mode`, `update_player_keys_for_terminal`, `update_player_for_terminal_mode`, `update_player_physics_variables`, `FLOOR`, `handle_player_in_vacuum`, `ReplenishPlayerOxygen`, `start_extravision_effect`, `PLAYER_IS_DEAD`, `revive_player`, `update_player_weapons`, `update_action_key`, `swipe_nearby_items`, `cause_polygon_damage`, `update_player_teleport`, `update_player_media`, `set_player_shapes`, `mark_oxygen_display_as_dirty`
- Notes: Netdead players (action_flags==0xffffffff) trigger disconnection message and detonation; predictive mode skips non-essential state updates

### damage_player
- Signature: `void damage_player(short monster_index, short aggressor_index, short aggressor_type, struct damage_definition *damage, short projectile_index)`
- Purpose: Apply damage to player, track stats, trigger death if needed
- Inputs: monster_index (player's monster slot), aggressor_index (source), aggressor_type (unused), damage (damage_definition with type/amount), projectile_index (for callbacks)
- Outputs/Return: None
- Side effects: Decrements suit_energy or suit_oxygen; updates damage records; may call kill_player; triggers fade effects
- Calls: `get_player_data`, `get_monster_data`, `calculate_damage`, `play_object_sound`, `kill_player`, `player_killed_player`, `Console::report_kill`, `L_Call_Player_Killed`, `start_fade`, `mark_shield_display_as_dirty`, `abort_terminal_mode`
- Notes: Invincibility powerup blocks all damage except player_settings.Vulnerability type; oxygen drain handled separately from energy; pegs values to 0/INT16_MAX to prevent overflow

### revive_player
- Signature: `void revive_player(short player_index)`
- Purpose: Resurrect dead player at spawn location with full health/oxygen
- Inputs: player_index (player to revive)
- Outputs/Return: None
- Side effects: Creates new monster/object at random spawn; resets physics, animations, weapons; restores max energy/oxygen; clears dead status flags
- Calls: `get_player_data`, `get_monster_data`, `calculate_player_team`, `get_random_player_starting_location_and_facing`, `remove_parasitic_object`, `turn_object_to_shit`, `new_map_object`, `attach_parasitic_object`, `initialize_player_physics_variables`, `give_player_initial_items`, `set_player_shapes`, `try_and_strip_player_items`, `update_interface`, `ChaseCam_Reset`, `ResetFieldOfView`, `L_Call_Player_Revived`
- Notes: Clears all transient flags except teleportation flags; runs callbacks for level scripts

### kill_player
- Signature: `static void kill_player(short player_index, short aggressor_player_index, short action)`
- Purpose: Kill player, set death animation, apply respawn penalty
- Inputs: player_index, aggressor_player_index (or NONE if monster/suicide), action (dying animation type)
- Outputs/Return: None
- Side effects: Discharges weapons; sets monster action to dying state; makes torso invisible; sets dead flag; starts reincarnation delay countdown
- Calls: `get_player_data`, `get_monster_data`, `discharge_charged_weapons`, `initialize_player_weapons`, `monster_died`, `set_player_dead_shape`, `kill_player_physics_variables`
- Notes: Applies game penalties if _dying_is_penalized or _suicide_is_penalized flags set; reincarnation_delay = MINIMUM (1 sec) + NORMAL (10 sec if dying penalized) + SUICIDE (15 sec if self-inflicted)

### set_player_shapes
- Signature: `static void set_player_shapes(short player_index, bool animate)`
- Purpose: Update player sprite animations based on current action, weapon, and powerup state
- Inputs: player_index, animate (if true, advance animation frames)
- Outputs/Return: None
- Side effects: Changes legs/torso shape descriptors; updates transfer modes (invisibility, invincibility effects); resets animation phase on shape change
- Calls: `get_player_data`, `get_monster_data`, `get_player_transfer_mode`, `get_player_weapon_mode_and_type`, `animate_object`, `GET_OBJECT_ANIMATION_FLAGS`, `SET_PLAYER_TOTALLY_DEAD_STATUS`, `remove_dead_player_items`
- Notes: Selects torso shape based on weapon and state (idle/charging/firing); legs driven by player->variables.action

### update_player_media
- Signature: `static void update_player_media(short player_index)` (not in provided excerpt but referenced)
- Purpose: Handle player movement through liquids (water, lava, sewage) with sound/splash effects
- Inputs: player_index
- Outputs/Return: None
- Side effects: Plays media sounds; triggers step sounds for wading
- Calls: Media and sound query functions
- Notes: Inferred from function references; manages viscosity and visual tint transitions

### get_player_transfer_mode
- Signature: `static void get_player_transfer_mode(short player_index, short *transfer_mode, short *transfer_period)`
- Purpose: Determine visual rendering mode (invisibility effect, invincibility flash, teleport fold, etc.)
- Inputs: player_index
- Outputs/Return: transfer_mode (blend/effect type), transfer_period (animation period in ticks)
- Side effects: None
- Calls: `get_player_data`, `PLAYER_IS_TELEPORTING`, `PLAYER_IS_INTERLEVEL_TELEPORTING`, `View_DoInterlevelTeleportOutEffects`, `View_DoInterlevelTeleportInEffects`
- Notes: Implements flashing logic when powerup about to expire; handles fold_in/fold_out for teleportation

### decode_hotkeys
- Signature: `void decode_hotkeys(ModifiableActionQueues& action_queues)`
- Purpose: Decode three-button weapon hotkey sequences (fire forward + fire right simultaneously to select weapon 1-9)
- Inputs: action_queues (mutable reference)
- Outputs/Return: None
- Side effects: Modifies action_flags to suppress/pass through cycle flags; sets player->hotkey if valid sequence decoded
- Calls: `peekActionFlags`, `get_player_data`, `modifyActionFlags`
- Notes: hotkey_sequence state machine: 0x03 (both cycle flags) ΓåÆ 0x0c (waiting for third press) ΓåÆ hotkey value or reset; dead players skip hotkey assignment if film_profile.hotkey_fix set

### unpack_player_data / pack_player_data
- Signature: `uint8 *unpack_player_data(uint8 *Stream, player_data *Objects, size_t Count)` / `uint8 *pack_player_data(...)`
- Purpose: Serialize/deserialize player state to/from binary stream for save games and network transmission
- Inputs: Stream (byte buffer pointer), Objects (player data array), Count (number of players)
- Outputs/Return: Updated stream pointer after operation
- Side effects: Reads from/writes to stream; zeroes hotkey_sequence and other transient fields on unpack
- Calls: `StreamToValue`, `StreamToBytes`, `StreamToList`, `StreamToPhysVars`, `StreamToDamRec`, `ValueToStream`, `BytesToStream`, `ListToStream`, `PhysVarsToStream`, `DamRecToStream`
- Notes: Fixed-size per-player record; hotkey_sequence reset because not serialized

### parse_mml_player / reset_mml_player
- Signature: `void parse_mml_player(const InfoTree& root)` / `void reset_mml_player(void)`
- Purpose: Load/reload player configuration from XML; save/restore original values for reset
- Inputs: root (InfoTree XML parser object)
- Outputs/Return: None
- Side effects: Overwrites global player_settings, player_initial_items, damage_response_definitions, player_shapes, etc.; allocates backup storage on first call
- Calls: `read_attr_bounded`, `read_attr`, `read_indexed`, `read_fixed`, `children_named`, Various parse/format helper methods
- Notes: Supports hierarchical XML with subtags for items, damage responses, powerups, shapes; MML allows complete customization of gameplay parameters

## Control Flow Notes
- **Initialization**: `initialize_players()` ΓåÆ `allocate_player_memory()` sets up global structures
- **Game startup**: `new_player()` called per player start ΓåÆ `recreate_player()` creates monster/object/physics
- **Main loop**: Each frame calls `update_players()` which:
  - Dequeues input actions from ActionQueues
  - Updates physics/location/animation
  - Handles powerup durations, oxygen depletion, environment effects
  - Dead players check for respawn trigger; live players pick up items, fire weapons
  - Updates terminal mode, teleportation, media transitions
- **Damage path**: Damage system ΓåÆ `damage_player()` ΓåÆ may call `kill_player()` ΓåÆ sets death animation
- **Revival**: Player presses action trigger ΓåÆ `kill_player()` checks reincarnation delay ΓåÆ calls `revive_player()` if eligible
- **Exit level**: `recreate_player()` called on level load to reset location/animation
- **Network**: `pack_player_data()` serializes state for transmission; `unpack_player_data()` deserializes on receive

## External Dependencies
- **map.h**: polygon_data, object_data, damage_definition, map object functions (new_map_object, translate_map_object, etc.), world geometry queries
- **monsters.h / monster_definitions.h**: monster_data, MONSTER_IS_PLAYER, monster_definition, damage types, monster flags
- **weapons.h**: weapon management, update_player_weapons, initialize_player_weapons, discharge_charged_weapons, player weapon mode queries
- **items.h**: item type constants, item functions (swipe_nearby_items, give/drop items)
- **projectiles.h**: detonate_projectile (for netdead effect)
- **SoundManager.h**: play_object_sound, sound effect enums
- **interface.h**: collection management, screen_printf, update_interface, mark_display_dirty functions
- **network.h / network_games.h**: multiplayer game state, player_killed_player callback
- **lua_script.h**: L_Call_Player_Damaged, L_Call_Player_Killed, L_Call_Player_Revived (Lua event callbacks)
- **ChaseCam.h**: ChaseCam_Reset for third-person view handling
- **ActionQueues.h**: dequeueActionFlags, modifyActionFlags, action queue management
- **InfoTree.h**: XML configuration parsing infrastructure
