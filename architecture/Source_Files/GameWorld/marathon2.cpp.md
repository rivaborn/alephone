# Source_Files/GameWorld/marathon2.cpp

## File Purpose

Central game loop and world state manager for the Marathon/Aleph One engine. Handles main tick simulation, action queue coordination, player movement prediction, level transitions, and trigger logic.

## Core Responsibilities

- **Main game loop** (`update_world`) ΓÇö orchestrates all per-tick entity and physics updates
- **Action queue management** ΓÇö merges input from real players, Lua scripts, and prediction systems
- **Client-side prediction** ΓÇö speculatively advances game state ahead of network confirmation, then rolls back when real updates arrive
- **Level transitions** ΓÇö `entering_map` and `leaving_map` coordinate resource loading/unloading and initialization
- **Polygon-based triggers** ΓÇö detects when entities cross boundaries and activates lights, platforms, monsters, items
- **Game state validation** ΓÇö calculates level completion conditions (extermination, exploration, retrieval, repair, rescue)
- **Damage calculations** ΓÇö applies difficulty modifiers and randomness

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| ModifiableActionQueues | class | Queue of per-player action flags (movement, firing, etc.); supports enqueue/dequeue/peek |
| player_data | struct | Player position, facing, weapons, health, animation state |
| monster_data | struct | Monster AI state, target, vitality, movement mode |
| object_data | struct | Generic entity (scenery, projectile, effect); contains shape, animation, transfer mode |
| polygon_data | struct | Map cell with type, height, media, sound sources, adjacent geometry |
| damage_definition | struct | Damage amount, type, flags, scaling for difficulty |
| dynamic_data | struct | Runtime tick count, entity counts, civilian casualties, random seed |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| GameQueue | ModifiableActionQueues* | static | Intermediate queue; receives action flags from real input or Lua, feeds main update |
| sMostRecentFlagsForPlayer | uint32\[\] | static | Last-known action flags per player; used to extrapolate player movement during prediction |
| sPredictedTicks | size_t | static | How many ticks ahead prediction has simulated |
| sPredictionWanted | bool | static | Flag to enable client-side prediction |
| sSavedPlayer* / sSavedMonster* / sSavedObject* | various arrays | static | Saved game state snapshots before entering prediction mode (for rollback) |
| sSavedTickCount, sSavedRandomSeed | int32, uint16 | static | Sanity checks to detect divergence during prediction |

## Key Functions / Methods

### initialize_marathon
- **Signature:** `void initialize_marathon(void)`
- **Purpose:** One-time engine initialization at startup
- **Side effects:** Builds trig tables; allocates map, pathfinding, texture memory; initializes weapon, window, scenery, and item subsystems; creates GameQueue; conditionally initializes OpenGL
- **Calls:** `build_trig_tables()`, `allocate_map_memory()`, `allocate_pathfinding_memory()`, `allocate_texture_tables()`, `initialize_weapon_manager()`, `initialize_game_window()`, `initialize_scenery()`, `initialize_items()`, `OGL_Initialize()`

### update_world
- **Signature:** `std::pair<bool, int16> update_world(void)`
- **Purpose:** Main tick handler; advances world by one or more ticks if input and network allow
- **Outputs:** `(bool, int16)` ΓÇö whether state changed (redraw?) and number of real ticks elapsed
- **Side effects:** Increments tick_count; updates all entities, Lua callbacks, fades, interface; may perform prediction
- **Calls:** Queues overlay functions; `update_world_elements_one_tick()`; `enter_predictive_mode()`, `exit_predictive_mode()`; `update_interface()`, `update_fades()`, networking and Lua handlers
- **Notes:** Returns early if network world update not ready; stops advancing when heartbeat/net time limit hit; performs prediction loop after real updates if enabled

### update_world_elements_one_tick
- **Signature:** `static int update_world_elements_one_tick(bool& call_postidle)`
- **Purpose:** Execute one tick of entity and physics updates
- **Outputs:** Enum ΓÇö `kUpdateNormalCompletion`, `kUpdateGameOver`, or `kUpdateChangeLevel`
- **Side effects:** Updates lights, media, platforms, control panels, players, projectiles, monsters, effects, scenery, ephemera, interpolated world; calls Lua `Idle` and networking
- **Calls:** `update_lights()`, `update_medias()`, `update_platforms()`, `update_players()`, `move_projectiles()`, `move_monsters()`, `update_effects()`, `check_level_change()`, `game_is_over()`

### entering_map
- **Signature:** `bool entering_map(bool restoring_saved)`
- **Purpose:** Initialize level after geometry/resources loaded; set up monsters, paths, sounds, Lua, networking
- **Inputs:** `restoring_saved` ΓÇö if true, skip Pfhortran (Lua) init
- **Outputs:** Success flag
- **Side effects:** Marks and loads shape/sound collections; syncs network; loads sounds patches; initializes monsters, net game, and Lua; stops fades; sets runtime flags
- **Calls:** `initialize_monsters_for_new_level()`, `load_collections()`, `load_all_monster_sounds()`, `load_all_game_sounds()`, `NetSync()`, `initialize_net_game()`, `L_Call_Init()`, `init_interpolated_world()`
- **Notes:** Can call `leaving_map()` on failure

### leaving_map
- **Signature:** `void leaving_map(void)`
- **Purpose:** Cleanup and save state before exiting level
- **Side effects:** Removes projectiles/effects; marks collections for unload; calls Lua cleanup; uploads stats; closes Lua; stops music/sounds; deactivates console
- **Calls:** `remove_all_projectiles()`, `remove_all_nonpersistent_effects()`, `L_Call_Cleanup()`, `CloseLuaScript()`, `Music::instance()->StopLevelMusic()`, `SoundManager::instance()->StopAllSounds()`

### changed_polygon
- **Signature:** `void changed_polygon(short original_polygon_index, short new_polygon_index, short player_index)`
- **Purpose:** Trigger entity entry events (lights, platforms, monsters, items) based on polygon type
- **Side effects:** Activates nearby monsters/items, toggles lights, triggers platform entry/state changes, clears "must explore" flags
- **Calls:** `get_polygon_data()`, `activate_nearby_monsters()`, `trigger_nearby_items()`, `set_light_status()`, `platform_was_entered()`, `try_and_change_platform_state()`
- **Notes:** Dispatches on polygon type (trigger, platform, light trigger, etc.)

### calculate_damage
- **Signature:** `short calculate_damage(struct damage_definition *damage)`
- **Purpose:** Compute final damage given base, random, scale, and difficulty modifiers
- **Outputs:** Final damage amount
- **Side effects:** Calls `global_random()`
- **Notes:** Reduces alien damage at lower difficulty levels

### calculate_level_completion_state / calculate_classic_level_completion_state
- **Signatures:** `short calculate_level_completion_state(void)` / `short calculate_classic_level_completion_state(void)`
- **Purpose:** Determine if level completion conditions met (kill all aliens, explore, retrieve items, repair switches, rescue civvies)
- **Outputs:** `_level_unfinished`, `_level_finished`, or `_level_failed`
- **Calls:** Mission flag checks; `live_aliens_on_map()`, `unretrieved_items_on_map()`, `untoggled_repair_switches_on_level()`
- **Notes:** `calculate_level_completion_state` allows Lua override; classic version enforces Marathon rules

### enter_predictive_mode / exit_predictive_mode
- **Signatures:** `static void enter_predictive_mode()` / `static void exit_predictive_mode()`
- **Purpose:** Save and restore partial game state to enable client-side movement prediction without network latency
- **Side effects:** Snapshots player, monster, object data (on entry); restores same (on exit), preserving interface flags
- **Calls:** `get_player_data()`, `get_monster_data()`, `get_object_data()`, `remove_object_from_polygon_object_list()`, `deferred_add_object_to_polygon_object_list()`, `perform_deferred_polygon_object_list_manipulations()`
- **Notes:** Only saves on first call (sPredictedTicks == 0); includes assertions to verify tick count and random seed consistency

### overlay_queue_with_queue_into_queue
- **Signature:** `static bool overlay_queue_with_queue_into_queue(ActionQueues* inBaseQueues, ActionQueues* inOverlayQueues, ActionQueues* inOutputQueues)`
- **Purpose:** Merge two action flag queues (base + overlay) into output, prioritizing overlay per-player
- **Outputs:** True if all players had flags in base queue; false otherwise
- **Side effects:** Dequeues base and overlay; enqueues to output
- **Notes:** Requires base queue completeness; dequeues base even if overridden by overlay

## Control Flow Notes

**Initialization ΓåÆ Level Entry ΓåÆ Main Loop ΓåÆ Level Exit**

1. **`initialize_marathon()`** ΓÇö Engine startup (called once)
2. **`entering_map()`** ΓÇö Load level; initialize monsters, paths, resources, Lua, networking
3. **`update_world()`** ΓÇö Repeated during gameplay:
   - Poll action queues (real input, Lua, prediction)
   - Call `update_world_elements_one_tick()` per available tick
   - Execute all entity updates (players ΓåÆ projectiles ΓåÆ monsters ΓåÆ effects)
   - Perform client-side prediction if enabled
   - Return redraw flag and elapsed real ticks
4. **`changed_polygon()`** ΓÇö Invoked per polygon boundary crossing; triggers context-sensitive events
5. **`leaving_map()`** ΓÇö Cleanup, save stats, unload resources

Prediction system runs **after** real updates: saves game state, simulates ahead using unconfirmed input, then rolls back on next confirmed update.

## External Dependencies

- **Geometry/Physics:** `map.h`, `world.h`, `flood_map.h`
- **Entities:** `monsters.h`, `projectiles.h`, `player.h`, `items.h`, `weapons.h`, `scenery.h`, `effects.h`
- **Systems:** `render.h`, `interface.h`, `network.h`, `network_games.h`, `SoundManager.h`, `Music.h`
- **Scripting:** `lua_script.h`, `lua_hud_script.h`
- **Input/Control:** `ActionQueues.h` (action queue system), input/output Lua and real player actions
- **Config/Profile:** `FilmProfile.h` (behavior flags for compatibility across game versions)
- **Misc:** `AnimatedTextures.h`, `ChaseCam.h`, `OGL_Setup.h`, `lightsource.h`, `media.h`, `platforms.h`, `fades.h`, `tags.h`, `motion_sensor.h`, `ephemera.h`, `interpolated_world.h`, `Plugins.h`, `Statistics.h`, `Console.h`, `Movie.h`, `game_window.h`, `screen.h`, `shell.h`, `SoundsPatch.h`

**Defined elsewhere:**
- `dynamic_world`, `static_world` ΓÇö global game/map state
- `GetGameQueue()`, `GetRealActionQueues()`, `GetLuaActionQueues()` ΓÇö action queue accessors
- `NetProcessMessagesInGame()`, `NetCheckWorldUpdate()`, `NetSync()`, `NetUpdateUnconfirmedActionFlags()`, `NetGetUnconfirmedActionFlag()`, `NetGetNetTime()` ΓÇö networking hooks
- `L_Call_Idle()`, `L_Call_PostIdle()`, `L_Call_Init()`, `L_Call_Cleanup()`, `L_Calculate_Completion_State()` ΓÇö Lua callbacks
- `update_lights()`, `update_medias()`, `update_platforms()`, `update_players()`, `move_projectiles()`, `move_monsters()`, `update_effects()` ΓÇö subsystem tick handlers
- `get_heartbeat_count()` ΓÇö frame-synchronization timer
