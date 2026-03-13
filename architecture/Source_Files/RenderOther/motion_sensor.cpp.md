# Source_Files/RenderOther/motion_sensor.cpp

## File Purpose
Implements the motion sensor HUD elementΓÇöa circular radar display showing nearby monsters, players, and network compass indicators. Tracks entity positions, manages visibility/fade animation, and renders colored blips for friendly, alien, and enemy targets. Supports software and OpenGL rendering backends.

## Core Responsibilities
- Entity tracking: maintain array of "blips" for monsters/players within sensor range
- Periodic scanning: rescan game world for entities within `MOTION_SENSOR_RANGE` 
- Position updates: shift entity history each frame and detect visibility changes
- Blip rendering: draw entity sprites with intensity/fade based on position history
- Network compass: display team beacon directions in multiplayer games
- Bitmap operations: sprite clipping and copying for software renderer
- Fade-out logic: gracefully remove entities by animating signal decay
- MML configuration: parse XML settings for frequency, range, scale, and monster display types

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `motion_sensor_definition` | struct | Settings: update/rescan frequency, range, scale factor |
| `entity_data` | struct | Tracked entity state: monster index, shape, position history (6 frames), visibility flags, removal delay |
| `region_data` | struct | Precomputed x-clip boundaries for circular sensor region (one per scanline) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `motion_sensor_player_index` | short | static | Index of player owning this sensor |
| `motion_sensor_side_length` | short | static | Sensor display dimensions (square) |
| `sensor_region` | region_data* | static | Precomputed circular clipping region array |
| `entities` | entity_data* | static | Array of up to 12 tracked entities |
| `network_compass_state` | short | static | Cached network compass state (NW/NE/SE/SW bits) |
| `mount_shape`, `virgin_mount_shapes`, `alien_shapes`, `friendly_shapes`, `enemy_shapes`, `compass_shapes` | shape_descriptor | static | HUD element shape references |
| `motion_sensor_changed` | bool | static | Dirty flag: display needs redraw |
| `ticks_since_last_update`, `ticks_since_last_rescan` | int32 | static | Frame counters for animation and scanning |
| `motion_sensor_settings` | motion_sensor_definition | static | Configurable parameters (frequency, range, scale) |
| `MonsterDisplays` | short[] | static | Per-monster-type display category (Friend/Alien/Enemy) |
| `OriginalMonsterDisplays`, `original_motion_sensor_settings` | ptrs | static | Backup for MML reset |

## Key Functions / Methods

### initialize_motion_sensor
- **Signature:** `void initialize_motion_sensor(shape_descriptor mount, shape_descriptor virgin_mounts, shape_descriptor aliens, shape_descriptor friends, shape_descriptor enemies, shape_descriptor compasses, short side_length)`
- **Purpose:** One-time setup: store shape descriptors, allocate entity array, allocate region clipping array, precompute sensor circular region.
- **Inputs:** HUD shape references, sensor size (pixels)
- **Outputs/Return:** None
- **Side effects:** Allocates dynamic arrays (`entities`, `sensor_region`); stores static shape references
- **Calls:** `new entity_data[]`, `new region_data[]`, `precalculate_sensor_region()`
- **Notes:** Must be called before shapes are loaded; entity array initialized with `flags = 0` (unused).

### reset_motion_sensor
- **Signature:** `void reset_motion_sensor(short player_index)`
- **Purpose:** Reset sensor state for a new level/player: restore virgin mount bitmap, clear entity list, zero compass state.
- **Inputs:** Player index
- **Outputs/Return:** None
- **Side effects:** Global state reset; `motion_sensor_player_index` set; bitmap copying
- **Calls:** `get_shape_bitmap_and_shading_table()`, `bitmap_window_copy()`, `MARK_SLOT_AS_FREE()` on all entities
- **Notes:** Skips bitmap copy if M1 shapes file (compatibility check).

### motion_sensor_scan
- **Signature:** `void motion_sensor_scan(void)`
- **Purpose:** Main per-tick update: rescan world if timer expired; update entity positions and fade.
- **Inputs:** None (uses global state)
- **Outputs/Return:** None
- **Side effects:** Updates `ticks_since_last_update`, `ticks_since_last_rescan`, entity positions, `motion_sensor_changed`
- **Calls:** `get_object_data()`, `get_monster_data()`, `guess_distance2d()`, `find_or_add_motion_sensor_entity()`, `erase_all_entity_blips()`
- **Notes:** Short-circuits if motion sensor disabled or inactive; iterates all monsters and adds those within range to sensor.

### render_motion_sensor (3 overloads: SW, OGL, Lua)
- **Signature (SW):** `void HUD_SW_Class::render_motion_sensor(short ticks_elapsed)`
- **Purpose:** Software renderer: restore virgin mount bitmap, redraw network compass, draw all blips via bitmap operations.
- **Inputs:** Elapsed ticks (unused)
- **Outputs/Return:** None
- **Side effects:** Bitmap manipulation on mount; calls `draw_network_compass()`, `draw_all_entity_blips()`
- **Calls:** `get_shape_bitmap_and_shading_table()`, `bitmap_window_copy()`, `draw_network_compass()`, `draw_all_entity_blips()`

### erase_all_entity_blips
- **Signature:** `void erase_all_entity_blips(void)`
- **Purpose:** Update all entity positions for current frame: shift position history, mark out-of-range/dead entities for removal, compute new position if active.
- **Inputs:** None (uses global state)
- **Outputs/Return:** None
- **Side effects:** Modifies entity position history, visibility flags, removal state; sets `motion_sensor_changed`
- **Calls:** `get_monster_data()`, `get_object_data()`, `transform_point2d()`
- **Notes:** Handles invisibility check, magnetic environment flicker; implements graceful fade-out by incrementing `remove_delay`.

### draw_all_entity_blips (SW/OGL variant) + Lua variant
- **Signature (SW):** `void HUD_Class::draw_all_entity_blips(void)` / `void HUD_Lua_Class::draw_all_entity_blips(void)`
- **Purpose:** Render all entity blips, iterating by intensity (fade stage) to ensure older/fainter blips drawn first.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Screen/buffer modifications; calls `draw_entity_blip()` for each visible entity at each intensity level
- **Calls:** `draw_entity_blip()`, `add_entity_blip()` (Lua), `clear_entity_blips()` (Lua)
- **Notes:** Inner loop over intensity allows layering: blips fade across 6 positions over multiple frames.

### find_or_add_motion_sensor_entity
- **Signature:** `static short find_or_add_motion_sensor_entity(short monster_index)`
- **Purpose:** Find existing entity by monster index or add new one if space available.
- **Inputs:** Monster index
- **Outputs/Return:** Entity index or `NONE` if no slot available
- **Side effects:** Modifies entity array (allocates new slot if needed); initializes entity state
- **Calls:** `get_monster_data()`, `get_object_data()`, `get_motion_sensor_entity_shape()`
- **Notes:** Skips adding if all 12 slots full; returns `NONE` if entity not found and cannot add.

### get_motion_sensor_entity_shape
- **Signature:** `static shape_descriptor get_motion_sensor_entity_shape(short monster_index)`
- **Purpose:** Determine display shape (friendly/alien/enemy) based on monster type and player team.
- **Inputs:** Monster index
- **Outputs/Return:** Shape descriptor (friendly_shapes, alien_shapes, or enemy_shapes)
- **Side effects:** None
- **Calls:** `get_monster_data()`, `get_player_data()`, `monster_index_to_player_index()`
- **Notes:** For players: use team comparison and game type to choose friendly vs. enemy; for monsters: use `MonsterDisplays[]` lookup table.

### precalculate_sensor_region
- **Signature:** `static void precalculate_sensor_region(short side_length)`
- **Purpose:** Precompute x-clip boundaries for each y-scanline to form a circular sensor boundary.
- **Inputs:** Sensor dimension (side_length)
- **Outputs/Return:** None
- **Side effects:** Populates `sensor_region[]` with x0/x1 clip boundaries
- **Calls:** None
- **Notes:** Uses circle equation `x┬▓ + y┬▓ = r┬▓` to compute clipping; prevents blips from rendering outside circular boundary.

### Bitmap copy functions
Three helper functions for software rendering:
- **`bitmap_window_copy`**: Copy rectangular region from source to destination (used to restore virgin mount).
- **`clipped_transparent_sprite_copy`**: Copy sprite with transparent-pixel skipping, clipped to circular sensor boundary.
- **`unclipped_solid_sprite_copy`**: Copy sprite without clipping (used for compass).

## Control Flow Notes
- **Init phase:** `initialize_motion_sensor()` called once on engine startup.
- **Per-level:** `reset_motion_sensor()` called when entering a map.
- **Per-frame (update):** `motion_sensor_scan()` called each tick to update entity positions and rescan world.
- **Per-frame (render):** `render_motion_sensor()` called by HUD renderer (only if `motion_sensor_has_changed()` returns true or on every frame for OGL/Lua variants).
- **Fade-out:** Entities marked "being removed" animate their `remove_delay` counter until all position history slots expire, then are freed.

## External Dependencies
- **Map/World:** `map.h` (objects, polygons, world coordinates), `monsters.h` (monster_data), `player.h` (player_data)
- **Rendering:** `render.h` (view_data), `interface.h` (shape descriptors, bitmap_definition, shape lookup functions)
- **Network:** `network_games.h` (get_network_compass_state)
- **HUD Renderers:** `HUDRenderer_SW.h`, `HUDRenderer_OGL.h`, `HUDRenderer_Lua.h` (base classes and rendering methods)
- **Config:** `InfoTree.h` (XML parsing for MML)
- **Math/Utility:** `cseries.h` (standard types), `<math.h>`, `<string.h>`, `<stdlib.h>`

**Defined elsewhere:** `get_object_data()`, `get_monster_data()`, `get_player_data()`, `guess_distance2d()`, `transform_point2d()`, `get_shape_bitmap_and_shading_table()`, `get_monster_index_to_player_index()`, `MotionSensorActive` (external bool), `m1_solo_player_in_terminal()`, shape-related accessors.
