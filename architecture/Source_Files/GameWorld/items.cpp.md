# Source_Files/GameWorld/items.cpp

## File Purpose
Manages item lifecycle in the game world: creation, placement, pickup mechanics, inventory updates, and animation. Handles both regular items (weapons, ammunition) and special item types (powerups, balls, team-specific items).

## Core Responsibilities
- Create and place new items in the world via `new_item()`
- Handle item pickup logic and inventory updates via `try_and_add_player_item()`
- Manage item visibility and collection (network-only, single-player filtering)
- Animate item shapes each frame via `animate_items()`
- Trigger hidden items to appear in specific zones via `trigger_nearby_items()`
- Support nearby item auto-pickup ("swiping") via `swipe_nearby_items()`
- Parse and manage MML-based item configuration
- Provide item metadata accessors (kind, shape, name)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `item_definition` | struct | Configurable properties: shape, kind, max count, names, environment restrictions |
| `object_data` | struct | Runtime object state; items use owner=`_object_is_item`, permutation=item type |
| `player_data` | struct | Player inventory stored as `items[NUMBER_OF_ITEMS]` array (counts) |
| `object_location` | struct | 3D position, polygon, facing for item placement |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `item_definitions` | `item_definition[]` | extern (global) | Master definitions for all item types; may be modified by MML |
| `original_item_definitions` | `item_definition*` | static | Backup copy of item definitions before MML overrides; used by reset |
| `objects` | `object_data[]` | extern (global) | All map objects; items identified by owner bits |
| `dynamic_world` | `dynamic_data*` | extern (global) | Game state: player count, game type, difficulty |
| `static_world` | `static_data*` | extern (global) | Map state: environment flags, ball_in_play |

## Key Functions / Methods

### new_item
- Signature: `short new_item(object_location *location, short type)`
- Purpose: Create and place a new item object in the game world.
- Inputs: location (3D position, polygon), item type ID
- Outputs/Return: object index (NONE if creation failed)
- Side effects: Calls `new_map_object()`, sets object flags and permutation, updates placement tracking, triggers Lua callback `L_Call_Item_Created()`. Plays sound for balls. Filters out network-only items in single-player, single-player items in multiplayer.
- Calls: `get_item_definition()`, `new_map_object()`, `get_object_data()`, `object_was_just_added()`, `L_Call_Item_Created()`
- Notes: Network/single-player filtering happens here; ball state tracked in `static_world->ball_in_play`; invisibility flag set for filtered-out items.

### try_and_add_player_item
- Signature: `bool try_and_add_player_item(short player_index, short type)`
- Purpose: Attempt to add an item to player's inventory; handles powerups, balls, and normal items differently.
- Inputs: player index, item type
- Outputs/Return: success (bool)
- Side effects: Increments player inventory count (respecting max), processes powerups, handles ball game logic (CTF/Rugby scoring), queues weapon reload, marks inventory dirty, plays pickup sound, calls Lua `L_Call_Got_Item()`, starts screen fade for local player.
- Calls: `get_item_definition()`, `get_player_data()`, `legal_player_powerup()`, `process_player_powerup()`, `object_was_just_destroyed()`, `Sound_GotPowerup()`, `find_player_ball_color()`, `get_polygon_data()`, `process_new_item_for_reloading()`, `mark_player_inventory_as_dirty()`, `Sound_GotItem()`, `L_Call_Got_Item()`, `start_fade()`, `SoundManager::instance()->PlaySound()`
- Notes: Special handling for CTF/Rugby ball logic (check if own-team ball in base, score on enemy base). Powerups destroy the object immediately. Normal items stack up to definition's max count.

### swipe_nearby_items
- Signature: `void swipe_nearby_items(short player_index)`
- Purpose: Auto-pickup items within arm's reach of player.
- Inputs: player index
- Outputs/Return: none
- Side effects: Searches nearby polygons, calls `get_item()` for each reachable visible item, removes picked-up items.
- Calls: Dispatcher to either `a1_swipe_nearby_items()` or `m2_swipe_nearby_items()` based on `film_profile.swipe_nearby_items_fix` flag
- Notes: Two implementations exist for compatibility (A1 vs M2 behavior). Range = `MAXIMUM_ARM_REACH = 3*WORLD_ONE_FOURTH`. Checks line solidity, platform obstruction, and vertical reach.

### a1_swipe_nearby_items / m2_swipe_nearby_items
- Signature: `static void a1_swipe_nearby_items(short player_index)` / `static void m2_swipe_nearby_items(short player_index)`
- Purpose: Internal implementations of item swiping with different polygon neighbor-checking logic.
- Inputs: player index
- Outputs/Return: none
- Side effects: Iterates neighbor polygons, calls `get_item()` to pick up visible items within reach.
- Calls: `get_player_data()`, `get_monster_data()`, `get_object_data()`, `get_polygon_data()`, `get_map_indexes()`, `guess_distance2d()`, `get_monster_dimensions()`, `test_item_retrieval()`, `get_item()`
- Notes: A1 version includes kludgy neighbor search starting from -1 to include current polygon; M2 version is simpler direct neighbor iteration.

### trigger_nearby_items
- Signature: `void trigger_nearby_items(short polygon_index)`
- Purpose: Activate (unhide) previously invisible items in all reachable polygons via flood fill.
- Inputs: polygon index (trigger origin)
- Outputs/Return: none
- Side effects: Uses `flood_map()` to find reachable polygons, calls `teleport_object_in()` on invisible items.
- Calls: `flood_map()`, `get_polygon_data()`, `get_object_data()`, `get_item_shape()`, `teleport_object_in()`
- Notes: Only unhides items with valid shapes; checks `OBJECT_IS_INVISIBLE()` and `permutation != NONE`.

### animate_items
- Signature: `void animate_items(void)`
- Purpose: Update animation frame for all visible items each tick.
- Inputs: none
- Outputs/Return: none
- Side effects: Iterates all objects, randomizes shape sequence once (if not animated), calls `animate_object()` for animated items.
- Calls: `get_object_data()`, `get_item_kind()`, `get_item_definition()`, `get_shape_animation_data()`, `randomize_object_sequence()`, `animate_object()`
- Notes: Uses `object->facing >= 0` as a flag to track whether randomization has been done; set to NONE after first randomize. Handles both animated and non-animated items.

### get_item
- Signature: `static bool get_item(short player_index, short object_index)`
- Purpose: Remove item object from world and add to player inventory.
- Inputs: player index, object (item) index
- Outputs/Return: success (bool)
- Side effects: Calls `try_and_add_player_item()`, removes object from map if successful.
- Calls: `get_object_data()`, `try_and_add_player_item()`, `remove_map_object()`
- Notes: Helper for swiping; asserts object is an item.

### test_item_retrieval
- Signature: `static bool test_item_retrieval(short polygon_index1, world_point3d *location1, world_point3d *location2)`
- Purpose: Verify line-of-sight retrieval path between item and player (raycast-like check).
- Inputs: starting polygon, player location, item location
- Outputs/Return: whether retrieval is valid (bool)
- Side effects: Traces line from player to item, checking for solid lines and moving platforms.
- Calls: `find_line_crossed_leaving_polygon()`, `find_adjacent_polygon()`, `get_line_data()`, `get_polygon_data()`, `get_platform_data()`
- Notes: Fails if solid line blocks path or moving platform in way.

### parse_mml_items
- Signature: `void parse_mml_items(const InfoTree& root)`
- Purpose: Parse XML configuration for item definitions (MML format).
- Inputs: InfoTree root (XML parsed tree)
- Outputs/Return: none
- Side effects: Backs up original definitions once, overwrites `item_definitions[]` with parsed values (names, max counts, shapes, difficulty overrides).
- Calls: Indirect via InfoTree methods (`read_indexed()`, `read_attr()`, `read_shape()`)
- Notes: Per-difficulty max counts can override base max count.

### reset_mml_items
- Signature: `void reset_mml_items()`
- Purpose: Restore original item definitions, discarding MML changes.
- Inputs: none
- Outputs/Return: none
- Side effects: Copies `original_item_definitions` back into `item_definitions`, frees backup.
- Calls: `free()`
- Notes: Idempotent if called twice (checks `original_item_definitions != NULL`).

### Supporting accessor functions
- **`get_item_definition(type)`**: Returns definition for item type; bounds-checked.
- **`get_item_definition_external(type)`**: Non-inlined wrapper for external callers.
- **`get_item_kind(item_id)`**: Returns item kind enum (powerup, weapon, ball, etc.).
- **`get_item_shape(item_id)`**: Returns shape descriptor for item.
- **`get_item_name(buffer, item_id, plural)`**: Formats localized item name into buffer.
- **`find_player_ball_color(player_index)`**: Searches player's items array for ball item, returns color (0ΓÇô7) or NONE.
- **`unretrieved_items_on_map()`**: Scans all objects for uncollected `_item` type items.
- **`item_valid_in_current_environment(item_type)`**: Checks environment flags against item restrictions.
- **`count_inventory_lines(player_index)`**: Counts distinct item types plus group headers for UI rendering.
- **`calculate_player_item_array(player_index, type, ...)`**: Builds array of items of specific kind in player's inventory.

## Control Flow Notes
- **Initialization**: `initialize_items()` is empty; definitions loaded externally via MML or defaults.
- **Frame update**: `animate_items()` called each tick to update item frame/sequence.
- **Pickup**: When player enters polygon, `swipe_nearby_items()` is called; each nearby item checked via `get_item()`, which calls `try_and_add_player_item()`.
- **Item spawning**: `new_item()` called at level load or via scripting; filters applied based on game mode.
- **Powerups**: Trigger special behavior (healing, invisibility, etc.) via `process_player_powerup()`; do not stack in inventory.
- **Shutdown**: No explicit shutdown; MML reset via `reset_mml_items()`.

## External Dependencies
- **map.h**: object data, polygon data, line data, dynamic_world, static_world, map geometry functions
- **player.h**: player data, inventory arrays, item pickup sounds
- **monsters.h**: monster data access for player's monster index
- **weapons.h**: `process_new_item_for_reloading()` (weapon integration)
- **SoundManager.h**: `PlaySound()`, sound index accessors (pickup sounds, powerup sounds)
- **fades.h**: `start_fade()` (screen flash on pickup)
- **interface.h**: collection marking, shape animation data
- **item_definitions.h**: `item_definitions[]` global array, `item_definition` struct
- **flood_map.h**: `flood_map()` (polygon reachability)
- **lua_script.h**: `L_Call_Item_Created()`, `L_Call_Got_Item()` (Lua hooks)
- **InfoTree.h**: XML parsing for MML configuration
- **FilmProfile.h**: `film_profile.swipe_nearby_items_fix` (compatibility flag)
- **network_games.h**: game type/option checks (CTF, Rugby, multiplayer)
