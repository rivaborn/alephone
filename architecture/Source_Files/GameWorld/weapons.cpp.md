# Source_Files/GameWorld/weapons.cpp

## File Purpose
Implements the weapon system for the Aleph One game engine, managing player weapon state machines, firing, reloading, animation, and shell casing physics. Handles all aspects of weapon selection, switching, ammo management, and state transitions.

## Core Responsibilities
- Initialize and manage weapon system for all players
- Update weapon state machines each frame (charging, firing, reloading, animations)
- Handle weapon firing events and projectile creation
- Manage ammo counts and reload mechanics for primary/secondary triggers
- Implement shell casing physics and interpolation for rendering
- Support weapon selection and automatic weapon switching
- Handle two-fisted and dual-function weapon behaviors
- Serialize/deserialize weapon data for save games
- Parse and apply MML configuration for weapon constants

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `WeaponConstant` | struct | Configurable per-weapon constants (separation, variance, overload timing) |
| `trigger_data` | struct | Current state of a weapon trigger (firing state, ammo, animation sequence) |
| `weapon_data` | struct | State container for a single weapon with both trigger states |
| `player_weapon_data` | struct | All weapon and shell-casing data for a player |
| `shell_casing_data` | struct | Physics state for an ejected shell casing |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `player_weapons_array` | `player_weapon_data*` | static | Array of weapon data for all players |
| `weapon_constants` | `std::array<WeaponConstant, ...>` | static | Per-weapon configurable constants |
| `original_weapon_constants` | `std::vector<WeaponConstant>` | static | Backup for MML reset |
| `original_shell_casing_definitions` | `shell_casing_definition*` | static | Backup for MML reset |
| `original_weapon_ordering_array` | `int16*` | static | Backup for MML reset |
| `shell_casing_id` | `short` | static | Sequence counter for interpolation |

## Key Functions / Methods

### initialize_weapon_manager
- Signature: `void initialize_weapon_manager(void)`
- Purpose: One-time initialization of weapon system; allocates player weapons array
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates heap memory for `player_weapons_array`
- Calls: `malloc()`, `objlist_clear()`
- Notes: Called once at engine startup

### initialize_player_weapons_for_new_game
- Signature: `void initialize_player_weapons_for_new_game(short player_index)`
- Purpose: Reset player weapons when starting a new game
- Inputs: Player index
- Outputs/Return: None
- Side effects: Clears all weapon state, resets ammo, initializes shell casings
- Calls: `initialize_player_weapons()`, `initialize_shell_casings()`

### update_player_weapons
- Signature: `void update_player_weapons(short player_index, uint32 action_flags)`
- Purpose: Main per-frame weapon update; processes triggers, state transitions, animations
- Inputs: Player index, action flags (button states)
- Outputs/Return: None
- Side effects: Updates weapon state machine, fires weapons, plays sounds, updates ammo display
- Calls: Large internal state machine; calls fire_weapon, reload_weapon, etc.
- Notes: Complex state machine with 16+ weapon states; called once per game tick per player

### fire_weapon
- Signature: `static void fire_weapon(short player_index, short which_trigger, _fixed charged_amount, bool flail_wildly)`
- Purpose: Execute weapon firing; create projectiles and effects
- Inputs: Player index, trigger index, charge level, flailing flag
- Outputs/Return: None
- Side effects: Calls new_projectile(), plays firing sounds, updates ammo, creates shell casings
- Calls: `new_projectile()`, `play_weapon_sound()`, `new_shell_casing()`, SoundManager

### reload_weapon
- Signature: `static bool reload_weapon(short player_index, short which_trigger)`
- Purpose: Initiate reload sequence for a weapon trigger
- Inputs: Player index, trigger index
- Outputs/Return: bool (true if reload possible)
- Side effects: Changes weapon state to awaiting_reload, calculates reload timing
- Calls: `get_player_trigger_definition()`

### ready_weapon
- Signature: `static bool ready_weapon(short player_index, short weapon_index)` (not static, used externally)
- Purpose: Bring a weapon to ready state from NONE; inverse of lowering
- Inputs: Player index, weapon index
- Outputs/Return: bool (success)
- Side effects: Sets current_weapon, initiates raising animation, preloads ammo if needed
- Calls: `check_reload()`, `put_rounds_into_weapon()`

### change_to_desired_weapon
- Signature: `static void change_to_desired_weapon(short player_index)`
- Purpose: Transition from current weapon to desired weapon
- Inputs: Player index
- Outputs/Return: None
- Side effects: Updates weapon state, initiates raising animation for new weapon
- Calls: Internal state setup, ammo loading

### select_next_best_weapon
- Signature: `static void select_next_best_weapon(short player_index)`
- Purpose: Auto-select next best available weapon after current one is unusable
- Inputs: Player index
- Outputs/Return: None
- Side effects: Sets desired_weapon field
- Calls: Searches weapon list for valid weapons

### update_shell_casings
- Signature: `static void update_shell_casings(short player_index)`
- Purpose: Update physics of ejected shell casings each frame
- Inputs: Player index
- Outputs/Return: None
- Side effects: Updates velocity, position, removes casings that are out of bounds
- Calls: Shell casing physics calculations
- Notes: Uses gravity and friction values from `shell_casing_definition`

### get_shell_casing_display_data
- Signature: `static bool get_shell_casing_display_data(struct weapon_display_information *display, short index)`
- Purpose: Retrieve rendering data for a visible shell casing
- Inputs: Display info struct pointer, index
- Outputs/Return: bool (valid casing found), fills display struct
- Side effects: Updates animation frame, interpolation data
- Calls: `get_shape_animation_data()`

### parse_mml_weapons
- Signature: `void parse_mml_weapons(const InfoTree& root)`
- Purpose: Load weapon configuration from MML (Marathon Markup Language)
- Inputs: InfoTree root node
- Outputs/Return: None
- Side effects: Modifies `weapon_constants`, `shell_casing_definitions`, `weapon_ordering_array`; backs up originals first
- Calls: InfoTree accessor methods
- Notes: Supports hot-reload of weapon balance without engine restart

### Packing/Unpacking Functions
- **Signatures**: `unpack_player_weapon_data()`, `pack_player_weapon_data()`, `unpack_weapon_definition()`, `pack_weapon_definition()`, `unpack_m1_weapon_definition()`
- **Purpose**: Serialize weapon data to/from byte streams for save games and network sync
- **Side effects**: Reads/writes weapon state, definitions, and shell casing data
- **Notes**: `unpack_m1_weapon_definition()` handles Marathon 1 format conversion; includes compatibility flags

## Control Flow Notes
**Initialization**: `initialize_weapon_manager()` called once at engine startup; `initialize_player_weapons_for_new_game()` called when player spawns or level loads.

**Frame Update**: `update_player_weapons()` is the main entry point each game tick. It processes all state transitions via a large switch statement, handling 16 weapon states. State transitions occur when `trigger->phase` decrements to zero. Nested state machine for two-fisted weapons adds complexity.

**Weapon Firing Chain**: Trigger-down ΓåÆ `handle_trigger_down()` ΓåÆ state change to `_weapon_firing` ΓåÆ `fire_weapon()` ΓåÆ `new_projectile()` ΓåÆ shell casing creation. Automatic weapons can stay firing for multiple frames.

**Serialization**: Called when saving/loading games or synchronizing networked players. Data includes current ammo, fire counts, and animation sequences.

## External Dependencies
- **Includes**: `map.h`, `projectiles.h`, `player.h`, `SoundManager.h`, `interface.h`, `items.h`, `monsters.h`, `Packing.h`, `shell.h`, `weapon_definitions.h`, `InfoTree.h`
- **External symbols**: 
  - `weapon_definitions`, `shell_casing_definitions`, `weapon_ordering_array` (defined elsewhere, probably weapons.h)
  - `new_projectile()` (projectiles.c)
  - `SoundManager::instance()->PlaySound()` (sound system)
  - `get_player_data()`, `dynamic_world` (player/map system)
  - `mark_collection_for_loading/unloading()` (shape/resource system)
  - `get_item_kind()` (items.c)
