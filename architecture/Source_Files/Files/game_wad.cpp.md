# Source_Files/Files/game_wad.cpp

## File Purpose
Manages loading, saving, and persistence of game maps and save states from WAD (archive) files. Orchestrates level initialization, game setup, network map distribution, and save/restore functionality for the game engine.

## Core Responsibilities
- Map file management (set, load, validate by checksum)
- Level loading from WAD files with geometry and entity placement
- New game initialization with player setup and level entry
- Save game creation and restoration (full game state serialization)
- Level entry point querying (spawn locations, mission types)
- Network map synchronization and distribution
- Dynamic memory allocation for map structures based on loaded counts
- Packing/unpacking of game state for persistence

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `save_game_data` | struct | Metadata describing a saveable chunk (tag, size, load flags) |
| `revert_game_info` | struct | Snapshot for reverting to previous game state (used for player retry) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MapFileSpec` | FileSpecifier | static | Currently loaded map file |
| `file_is_set` | bool | static | Whether a map file is active |
| `PolygonListCopy` | vector\<polygon_data\> | static | Snapshot of polygons for export (initial state) |
| `PlatformListCopy` | vector\<platform_data\> | static | Snapshot of platforms for export (initial state) |
| `revert_game_data` | revert_game_info | static | Saved game context for reverting to previous level |
| `static_platforms` | vector\<static_platform_data\> | static | Platform snapshot used during export |
| `export_data[]` | save_game_data[] | static | Metadata array defining exportable chunks (points, polygons, platforms, etc.) |
| `save_data[]` | save_game_data[] | static | Metadata array defining all saveable chunks (full game state) |

## Key Functions / Methods

### set_map_file
- **Signature:** `void set_map_file(FileSpecifier& File, bool loadScripts)`
- **Purpose:** Set the current map file and load associated metadata/scripts
- **Inputs:** Map file specifier; whether to load level scripts
- **Outputs/Return:** None
- **Side effects:** Updates `MapFileSpec`, `file_is_set`; calls RunRestorationScript, loads plugin checksums; triggers level script loading
- **Calls:** `RunRestorationScript()`, `set_scenario_images_file()`, `get_current_map_checksum()`, `LoadLevelScripts()`, `clear_game_error()`

### load_level_from_map
- **Signature:** `bool load_level_from_map(short level_index)`
- **Purpose:** Load a map level from the current WAD file
- **Inputs:** Level index (or NONE to restore save game)
- **Outputs/Return:** Success flag
- **Side effects:** Allocates map structures; unpacks map geometry and entities; may restore saved game state if index==NONE
- **Calls:** `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `process_map_wad()`, `close_wad_file()`

### goto_level
- **Signature:** `bool goto_level(struct entry_point *entry, short number_of_players, player_start_data* player_start_information)`
- **Purpose:** Transition to a new level (handles script loading, networking, object placement, player recreation)
- **Inputs:** Target entry point; player count; player start data (NULL for level change, non-NULL for new game)
- **Outputs/Return:** Success flag
- **Side effects:** Calls `leaving_map()` for level change; runs level scripts; triggers network map sync or local load; places initial objects; initializes control panels
- **Calls:** `leaving_map()`, `ResetLevelScript()`, `RunLevelScript()`, `NetChangeMap()`, `load_level_from_map()`, `LoadSoloLua()`, `LoadReplayNetLua()`, `RunLuaScript()`, `entering_map()`, `place_initial_objects()`, `initialize_control_panels_for_level()`

### new_game
- **Signature:** `bool new_game(short number_of_players, bool network, struct game_data *game_information, struct player_start_data *player_start_information, struct entry_point *entry_point)`
- **Purpose:** Initialize a new game session (single or networked)
- **Inputs:** Player count; network flag; game and player setup data; initial level entry point
- **Outputs/Return:** Success flag
- **Side effects:** Resets global game state; sets random seed; initializes player list; loads map; creates players; sets up revert state
- **Calls:** `ResetPassedLua()`, `set_random_seed()`, `initialize_map_for_new_game()`, `goto_level()`, `create_players_for_new_game()`, `setup_revert_game_info()`, `reset_action_queues()`, `entering_map()`, `reset_motion_sensor()`, `ChaseCam_Initialize()`

### process_map_wad
- **Signature:** `bool process_map_wad(struct wad_data *wad, bool restoring_game, short version)` (declared, called from multiple sites)
- **Purpose:** Core loader: unpacks all map data from WAD chunk (geometry, objects, physics, scripting)
- **Inputs:** WAD data; restore-game flag; data version for compatibility
- **Outputs/Return:** Success flag
- **Side effects:** Allocates/populates all map structures; loads monsters, effects, projectiles, platforms; executes level scripts; handles physics model loading
- **Calls:** Many unpack_*_data functions; `RunScriptChunks()`; `get_dynamic_data_from_wad()`; `get_player_data_from_wad()`; `complete_loading_level()` or `complete_restoring_level()`

### build_save_game_wad
- **Signature:** `static struct wad_data *build_save_game_wad(struct wad_header *header, int32 *length)`
- **Purpose:** Serialize full game state into a WAD for saving
- **Inputs:** WAD header; output length pointer
- **Outputs/Return:** New WAD containing all saveable chunks
- **Side effects:** Allocates temporary packed-data buffers; queries all game state counts
- **Calls:** `create_empty_wad()`, `recalculate_map_counts()`, `tag_to_global_array_and_size()` (for each save_data chunk), `append_data_to_wad()`, `calculate_wad_length()`

### build_export_wad
- **Signature:** `static wad_data *build_export_wad(wad_header *header, int32 *length)`
- **Purpose:** Serialize map for export (initial state, not save state)
- **Inputs:** WAD header; output length pointer
- **Outputs/Return:** New WAD with exportable chunks
- **Side effects:** Snapshots and restores platform/polygon/line/side state; recalculates variable-elevation flags; handles platform state inference
- **Calls:** Similar to build_save_game_wad but uses export_data array and export_tag_to_global_array_and_size

### complete_loading_level
- **Signature:** `void complete_loading_level(short *_map_indexes, size_t map_index_count, uint8 *_platform_data, size_t platform_data_count, uint8 *actual_platform_data, size_t actual_platform_data_count, short version)`
- **Purpose:** Finalize level load: recalculate redundant data, add platforms, scenery; handle Marathon 1 compatibility
- **Inputs:** Map indices; platform data; version for format compatibility
- **Outputs/Return:** None
- **Side effects:** Loads redundant map data; adds platforms and scenery; sets up lightsource indices for Marathon 1; initializes scenery solidity
- **Calls:** `load_redundant_map_data()`, `scan_and_add_platforms()`, `scan_and_add_scenery()`, `guess_side_lightsource_indexes()`

### get_indexed_entry_point / get_entry_points
- **Signature:** `bool get_indexed_entry_point(struct entry_point *entry_point, short *index, int32 type)` / `bool get_entry_points(vector<entry_point> &vec, int32 type)`
- **Purpose:** Query available level entry points (spawn locations by game type)
- **Inputs:** Entry point to fill (or vector); filter type (single-player, coop, carnage, etc.)
- **Outputs/Return:** Success; populates entry_point(s)
- **Side effects:** Reads map directory data from WAD; converts Marathon 1 entry point flags if needed
- **Calls:** `open_wad_file_for_reading()`, `read_wad_header()`, `read_directory_data()` or `read_indexed_wad_from_file()`, `unpack_directory_data()`, `unpack_static_data()`

### allocate_map_for_counts
- **Signature:** `void allocate_map_for_counts(size_t polygon_count, size_t side_count, size_t endpoint_count, size_t line_count)`
- **Purpose:** Pre-allocate all map structure memory based on expected counts
- **Inputs:** Counts for geometry elements
- **Outputs/Return:** None
- **Side effects:** Resizes global EndpointList, LineList, SideList, PolygonList, AutomapLineList, AutomapPolygonList, MapIndexList; calls allocate_render_memory, allocate_flood_map_memory
- **Calls:** Resizing operations on global vectors; `allocate_render_memory()`, `allocate_flood_map_memory()`

### load_* helper functions
- **Signatures:** `void load_points(uint8 *points, size_t count)`, `load_lines()`, `load_sides()`, `load_polygons()`, `load_lights()`, `load_annotations()`, `load_objects()`, `load_media()`, `load_map_info()`, `load_ambient_sound_images()`, `load_random_sound_images()`, `load_terminal_data()`
- **Purpose:** Unpack individual map data chunks from byte streams
- **Inputs:** Packed byte data; count of items
- **Outputs/Return:** None (populate global structures)
- **Side effects:** Unpack and populate corresponding global lists; update dynamic_world counts
- **Notes:** Use helper functions `unpack_*_data()` and `StreamToValue()` for serialization

### tag_to_global_array_and_size / export_tag_to_global_array_and_size
- **Signature:** `static uint8 *tag_to_global_array_and_size(uint32 tag, size_t *size)` (and export variant)
- **Purpose:** Pack game state into a byte buffer for a specific WAD chunk type
- **Inputs:** Chunk tag ID; output size pointer
- **Outputs/Return:** Allocated byte array (or NULL if empty)
- **Side effects:** Allocates temporary buffer; queries global state counts and data
- **Calls:** Various `pack_*_data()` functions; `count_number_of_medias_used()`, `calculate_packed_terminal_data_length()`, etc.
- **Notes:** Large switch statement handling 30+ chunk types; caller must delete[] returned buffer

## Control Flow Notes

**Initialization (New Game):**
1. `new_game()` ΓåÆ `initialize_map_for_new_game()` ΓåÆ `goto_level()`
2. `goto_level()` ΓåÆ `load_level_from_map()` (or `NetChangeMap()` if networked)
3. `load_level_from_map()` ΓåÆ `process_map_wad()` ΓåÆ `complete_loading_level()`
4. `place_initial_objects()`, `entering_map()` finalize setup

**Level Transition:**
1. `goto_level()` ΓåÆ `leaving_map()` (cleanup)
2. Repeat initialization flow (but recreates players instead of creating new ones)

**Saving:**
- Standalone; `build_save_game_wad()` serializes all dynamic state
- Called by save game UI layer (not in this file)

**Restoration:**
- `load_level_from_map(level_index=NONE)` ΓåÆ `process_map_wad(restoring_game=true)`
- Unpacks player, object, monster, projectile, effect, platform, weapon, terminal state from save WAD

**Networking:**
- `get_map_for_net_transfer()` ΓåÆ `get_flat_data()` (compress map for network)
- `process_net_map_data()` ΓåÆ `inflate_flat_data()` ΓåÆ `process_map_wad()` (receive map from network)

## External Dependencies
- **Map/World:** map.h, world.h ΓÇö defines all geometry and entity structures
- **Game Systems:** monsters.h, projectiles.h, effects.h, player.h, platforms.h ΓÇö entity management
- **Networking:** network.h ΓÇö map sync, player data distribution
- **I/O:** FileHandler.h, wad.h, Packing.h ΓÇö file and data serialization
- **Scripting:** XML_LevelScript.h ΓÇö MML level script loading
- **UI:** interface.h, game_window.h, shell.h ΓÇö error reporting, shell interaction
- **Other:** lightsource.h, media.h, weapons.h, scenery.h, computer_interface.h, images.h, SoundManager.h, Plugins.h, ChaseCam.h, render.h, motion_sensor.h, Music.h, ephemera.h
- **Defined Elsewhere:** `process_map_wad()`, `entering_map()`, `leaving_map()`, `setup_revert_game_info()`, `recreate_players_for_new_level()`, `get_flat_data()`, `inflate_flat_data()` etc.
