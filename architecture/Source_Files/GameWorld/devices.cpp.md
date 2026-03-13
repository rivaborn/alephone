# Source_Files/GameWorld/devices.cpp

## File Purpose
Manages interactive control panels in the game worldΓÇöswitches, terminals, refuel stations, and save points. Handles panel initialization, player/projectile interaction, state synchronization with linked platforms and lights, and MML configuration support.

## Core Responsibilities
- Initialize control panel states based on linked platforms/lights at level start
- Update panel states during gameplay (refueling, save game cooldown)
- Detect player action-key targets (panels, platforms) within activation range
- Toggle panels via player action or projectile hit
- Synchronize panel visual state and associated sounds
- Handle specialized panels: terminals, pattern buffers (save points), refuel stations
- Parse and reset MML-based configuration (activation ranges, energy amounts, sounds)
- Invoke Lua script hooks on panel state changes

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `control_panel_definition` | struct | Static definition: class, graphics, sounds, activation item for one panel type |
| `control_panel_settings_definition` | struct | Runtime settings: activation reach distance/horizontal scale, energy caps and rates |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `control_panel_definitions[]` | struct array | static | One definition per control panel type; immutable after init |
| `control_panel_settings` | struct | global | Editable activation ranges and energy recharge parameters |
| `original_control_panel_definitions` | struct* | static | Backup for MML reset |
| `original_control_panel_settings` | struct* | static | Backup for MML reset |

## Key Functions / Methods

### initialize_control_panels_for_level
- **Signature:** `void initialize_control_panels_for_level(void)`
- **Purpose:** Set initial on/off state of all panels based on linked platform/light state.
- **Inputs:** None (reads from global map data)
- **Outputs/Return:** None; modifies side flags
- **Side effects:** Iterates all map sides, calls `SET_CONTROL_PANEL_STATUS`, `set_control_panel_texture()`
- **Calls:** `get_control_panel_definition()`, `get_light_status()`, `platform_is_on()`, `set_control_panel_texture()`
- **Notes:** Disables invalid panels (missing definition or missing platform index).

### update_control_panels
- **Signature:** `void update_control_panels(void)`
- **Purpose:** Handle continuous state updates (oxygen/energy recharge) for players using panels.
- **Inputs:** None (reads from global `players[]`, `dynamic_world->tick_count`)
- **Outputs/Return:** None; modifies player suit energy/oxygen
- **Side effects:** Increments player oxygen/energy if stationary; stops recharge if player moves or switches panels
- **Calls:** `change_panel_state()`, `play_control_panel_sound()`, `SoundManager::instance()->StopSound()`
- **Notes:** Pattern buffer logic distinguishes network vs. single-player save mechanics; double-click detection for overwrite saves.

### update_action_key
- **Signature:** `void update_action_key(short player_index, bool triggered)`
- **Purpose:** Handle player action-key press; dispatch to appropriate handler (platform, control panel).
- **Inputs:** player index, trigger flag
- **Outputs/Return:** None; invokes state changes
- **Side effects:** Calls `find_action_key_target()`, then routes to `player_touch_platform_state()` or `change_panel_state()`
- **Calls:** `find_action_key_target()`, `player_touch_platform_state()`, `change_panel_state()`

### change_panel_state
- **Signature:** `static void change_panel_state(short player_index, short panel_side_index)`
- **Purpose:** Toggle panel state and handle side effects (light/platform state, terminal entry, save game, refuel start/stop).
- **Inputs:** player index, side index of panel
- **Outputs/Return:** None; modifies global state
- **Side effects:** Updates panel flags, lights, platforms, player oxygen/energy, player terminal index, invokes Lua hooks
- **Calls:** `set_light_status()`, `try_and_change_platform_state()`, `enter_computer_interface()`, `save_game()`, `L_Call_*()` (Lua hooks)
- **Notes:** Different logic per panel class; refuel panels toggle player's association with that panel; pattern buffer requires cooldown to prevent spam.

### try_and_toggle_control_panel
- **Signature:** `void try_and_toggle_control_panel(short polygon_index, short line_index, short projectile_index)`
- **Purpose:** Handle projectile impact on a control panel; toggle if permitted.
- **Inputs:** polygon and line indices (to find adjacent side), projectile index
- **Outputs/Return:** None
- **Side effects:** Toggles panel state if `switch_can_be_toggled()` permits; may destroy panel if flagged
- **Calls:** `find_adjacent_side()`, `switch_can_be_toggled()`, `set_tagged_light_statuses()`, `try_and_change_tagged_platform_states()`, `set_light_status()`, `try_and_change_platform_state()`, `play_control_panel_sound()`, `L_Call_Projectile_Switch()`

### find_action_key_target
- **Signature:** `short find_action_key_target(short player_index, world_distance range, short *target_type, bool perform_panel_actions)`
- **Purpose:** Ray-cast from player in facing direction to find nearest interactive target (platform or control panel) within range.
- **Inputs:** player index, range limit, output pointer for target type, flag to perform audio/destruction
- **Outputs/Return:** Index of target (platform or side); `*target_type` set to `_target_is_platform` or `_target_is_control_panel`
- **Side effects:** If `perform_panel_actions=true`, plays error sound for unusable panels
- **Calls:** `ray_to_line_segment()`, `find_line_crossed_leaving_polygon()`, `get_polygon_data()`, `line_is_within_range()`, `platform_is_legal_player_target()`, `line_side_has_control_panel()`, `switch_can_be_toggled()`, `play_control_panel_sound()`
- **Notes:** Polygon-walking algorithm to trace ray; stops at first valid target or level boundary.

### line_side_has_control_panel
- **Signature:** `bool line_side_has_control_panel(short line_index, short polygon_index, short *side_index_with_panel)`
- **Purpose:** Check if a line's side (facing a given polygon) has a control panel.
- **Inputs:** line and polygon indices
- **Outputs/Return:** `true` if panel exists; `*side_index_with_panel` set to the side index
- **Side effects:** None
- **Calls:** `get_line_data()`, `get_side_data()`, `SIDE_IS_CONTROL_PANEL()` macro
- **Notes:** Handles both clockwise and counterclockwise polygon ownership.

### set_control_panel_texture
- **Signature:** `void set_control_panel_texture(struct side_data *side)`
- **Purpose:** Update side texture descriptor to active or inactive shape based on panel status.
- **Inputs:** side data pointer
- **Outputs/Return:** None; modifies `side->primary_texture.texture`
- **Side effects:** Updates texture descriptor
- **Calls:** `get_control_panel_definition()`, `BUILD_DESCRIPTOR()` macro

### switch_can_be_toggled
- **Signature:** `static bool switch_can_be_toggled(short side_index, bool player_hit, bool *should_destroy_switch)`
- **Purpose:** Determine if switch activation is legal (lighting, item requirement, attack mode restrictions).
- **Inputs:** side index, whether hit by player (vs. projectile), output flag for destruction
- **Outputs/Return:** `true` if toggle is permitted
- **Side effects:** None; possibly sets `*should_destroy_switch`
- **Calls:** `get_light_intensity()` (two variants for different intensity thresholds)

### play_control_panel_sound
- **Signature:** `static void play_control_panel_sound(short side_index, short sound_index, bool soft_rewind=false)`
- **Purpose:** Play activation, deactivation, or error sound for a panel.
- **Inputs:** side index, sound type enum, soft-rewind flag
- **Outputs/Return:** None; audio side effect
- **Side effects:** Sound playback
- **Calls:** `get_side_data()`, `get_control_panel_definition()`, `play_side_sound()`

### get_control_panel_definition
- **Signature:** `control_panel_definition *get_control_panel_definition(const short control_panel_type)`
- **Purpose:** Safe accessor to control panel definition; returns NULL if index out of bounds.
- **Inputs:** panel type (index)
- **Outputs/Return:** Pointer to definition or NULL
- **Side effects:** None
- **Calls:** `GetMemberWithBounds()` (bounds-checking template)

### parse_mml_control_panels
- **Signature:** `void parse_mml_control_panels(const InfoTree& root)`
- **Purpose:** Parse XML configuration to customize panel definitions, activation ranges, and sounds.
- **Inputs:** `InfoTree` node containing panel configuration
- **Outputs/Return:** None; modifies global state
- **Side effects:** Updates `control_panel_definitions[]` and `control_panel_settings`; backs up originals on first call
- **Calls:** `root.read_attr()`, `root.read_wu()`, `root.read_indexed()`, `root.children_named()`
- **Notes:** Supports per-panel customization (type, collection, shapes, sounds, item, pitch).

### reset_mml_control_panels
- **Signature:** `void reset_mml_control_panels()`
- **Purpose:** Restore original definitions and settings after MML overrides.
- **Inputs:** None
- **Outputs/Return:** None; restores global state
- **Side effects:** Deallocates backup memory
- **Calls:** None (memcpy-equivalent operations)

## Control Flow Notes

- **Level init** ΓåÆ `initialize_control_panels_for_level()` sets panel states based on initial linked platform/light positions.
- **Per-frame update** ΓåÆ `update_control_panels()` handles ongoing recharge logic.
- **Player input** ΓåÆ `update_action_key()` detects action key, calls `find_action_key_target()` to ray-cast, dispatches to `change_panel_state()`.
- **Projectile interaction** ΓåÆ `try_and_toggle_control_panel()` called via collision handler; toggles if allowed.
- **Config reload** ΓåÆ `parse_mml_control_panels()` called at game init; `reset_mml_control_panels()` during shutdown/reload.

## External Dependencies

- **map.h** ΓÇô `side_data`, `polygon_data`, `line_data`, `get_*_data()`, `dynamic_world`, map globals
- **monsters.h** ΓÇô `get_monster_data()`, `get_monster_dimensions()`, `monster_index`
- **player.h** ΓÇô `player_data`, `players[]`, player state and item management
- **platforms.h** ΓÇô `get_polygon_data()`, `platform_is_on()`, `try_and_change_platform_state()`, platform state
- **SoundManager.h** ΓÇô `SoundManager::instance()->StopSound()`
- **computer_interface.h** ΓÇô `enter_computer_interface()`, terminal mode
- **lightsource.h** ΓÇô `get_light_status()`, `set_light_status()`, `get_light_intensity()`
- **lua_script.h** ΓÇô `L_Call_*()` hooks (Lua callbacks)
- **InfoTree.h** ΓÇô MML XML parsing

## Notes

- Panels are tightly coupled to the map geometry (sides, lines, polygons) and linked resources (platforms, lights).
- Lua scripting integration allows mod authors to customize panel behavior and respond to state changes.
- Network save logic (pattern buffer) includes anti-spam double-click detection and overwrite prevention.
- MML support enables mod customization without code changes.
