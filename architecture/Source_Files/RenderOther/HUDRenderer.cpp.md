# Source_Files/RenderOther/HUDRenderer.cpp

## File Purpose
Implements the HUD rendering system for the Aleph One game engine (Marathon port). Renders player-facing interface elements including shields, oxygen, weapons, ammunition counts, inventory, player names, and network messages. Supports both standard HUD display and debug Lua texture palette visualization.

## Core Responsibilities
- Update and render all HUD elements each frame via dirty-flag optimization
- Render multi-level shield/energy bars with corresponding background textures
- Render oxygen bar with frame-throttled updates (2-tick delay)
- Display current weapon panel with multi-weapon variants and weapon names
- Render ammunition displays (both energy meter and discrete bullet grids)
- Maintain and render inventory panels with item names, quantities, and validity indicators
- Calculate and manage interface rectangle geometry and text layout
- Support Lua script texture palette debug visualization (grid layout of textures)
- Delegate platform-specific drawing operations to virtual methods (DrawShape, DrawText, FillRect, etc.)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `HUD_Class` | class | Base class providing HUD rendering interface; subclasses implement virtual drawing methods |
| `weapon_interface_data` | struct | Configuration for weapon panel display: shape, position, name layout, ammo configurations |
| `weapon_interface_ammo_data` | struct | Ammo display config: type (energy/bullets), screen position, grid dimensions, bullet shapes |
| `interface_state_data` | struct | Dirty flags tracking which HUD elements need redraw (shield, oxygen, weapon, ammo) |
| `screen_rectangle` | struct | Rectangular region on screen (defined elsewhere); has top, left, bottom, right |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `interface_state` | `interface_state_data` | extern global | Dirty flags for HUD elements; signals which subsystems need redraw |
| `weapon_interface_definitions[10]` | `weapon_interface_data[]` | extern global | Array of weapon panel configurations for all weapons |
| `delay_time` | `short` (static) | static in `update_suit_oxygen()` | Throttle oxygen bar redraws to every 2 seconds |

## Key Functions / Methods

### update_everything
- **Signature:** `bool HUD_Class::update_everything(short time_elapsed)`
- **Purpose:** Main frame update for all HUD elements; gateway for dirty-flag optimization
- **Inputs:** `time_elapsed` ΓÇö milliseconds since last frame (NONE = force redraw)
- **Outputs/Return:** `bool ForceUpdate` ΓÇö true if any element was redrawn
- **Side effects:** Sets `ForceUpdate` flag; calls all sub-update methods
- **Calls:** `LuaTexturePaletteSize()`, `shapes_file_is_m1()`, `update_motion_sensor()`, `update_inventory_panel()`, `update_weapon_panel()`, `update_ammo_display()`, `update_suit_energy()`, `update_suit_oxygen()`, `draw_message_area()`, `LuaTexturePaletteTexture()`, `DrawTexture()`, `FrameRect()`
- **Notes:** Special case: if Lua texture palette is active, draws debug grid of textures instead of standard HUD; does not call other update methods in that case

### update_suit_energy
- **Signature:** `void HUD_Class::update_suit_energy(short time_elapsed)`
- **Purpose:** Render shield/energy bar; selects bar texture based on energy level (single, double, triple)
- **Inputs:** `time_elapsed` (NONE = force redraw); checks `interface_state.shield_is_dirty`
- **Outputs/Return:** None
- **Side effects:** Sets `ForceUpdate` and clears `shield_is_dirty` flag
- **Calls:** `get_interface_rectangle()`, `draw_bar()`, `BUILD_DESCRIPTOR()`
- **Notes:** Maps energy multiples (1x, 2x, 3x PLAYER_MAXIMUM_SUIT_ENERGY) to different bar textures; clamps width proportionally

### update_suit_oxygen
- **Signature:** `void HUD_Class::update_suit_oxygen(short time_elapsed)`
- **Purpose:** Render oxygen bar with frame-rate throttling
- **Inputs:** `time_elapsed`; checks `interface_state.oxygen_is_dirty`
- **Outputs/Return:** None
- **Side effects:** Sets `ForceUpdate`; decrements static `delay_time`; clears `oxygen_is_dirty` flag
- **Calls:** `get_interface_rectangle()`, `draw_bar()`, `BUILD_DESCRIPTOR()`
- **Notes:** Redraws only if delay_time expires (every ~2 seconds) or dirty flag set; uses MIN() to cap displayed oxygen

### update_weapon_panel
- **Signature:** `void HUD_Class::update_weapon_panel(bool force_redraw)`
- **Purpose:** Render weapon display panel with weapon shape and name
- **Inputs:** `force_redraw` or checks `interface_state.weapon_is_dirty`
- **Outputs/Return:** None
- **Side effects:** Sets `ForceUpdate`; clears `weapon_is_dirty` flag; marks `ammo_is_dirty` for update
- **Calls:** `get_interface_rectangle()`, `FillRect()`, `get_player_desired_weapon()`, `DrawShapeAtXY()`, `getcstr()`, `get_item_name()`, `DrawText()`
- **Notes:** Handles multi-weapon variants (draws single + multi shapes, or multi only); checks item count to show correct variant; retrieves weapon name from string table

### draw_bar
- **Signature:** `void HUD_Class::draw_bar(screen_rectangle *rectangle, short width, shape_descriptor top_piece, shape_descriptor full_bar, shape_descriptor background_texture)`
- **Purpose:** Render a progress bar (shield/oxygen) with background, fill, and cap shapes
- **Inputs:** Rectangle bounds, fill width, three shape descriptors (cap, fill, background)
- **Outputs/Return:** None
- **Side effects:** Calls DrawShape() multiple times
- **Calls:** `DrawShape()`, `DrawShapeAtXY()`, `offset_rect()`
- **Notes:** Handles edge case where width < 2├ùTOP_OF_BAR_WIDTH by repositioning cap; uses src/dest rectangles for shape clipping

### draw_ammo_display_in_panel
- **Signature:** `void HUD_Class::draw_ammo_display_in_panel(short trigger_id)`
- **Purpose:** Render ammunition display as either energy meter or discrete bullet grid (depends on weapon config)
- **Inputs:** `trigger_id` ΓÇö _primary_interface_ammo or _secondary_interface_ammo
- **Outputs/Return:** None
- **Side effects:** Calls DrawShapeAtXY() and FillRect() for each bullet or energy segment
- **Calls:** `get_player_desired_weapon()`, `get_player_weapon_ammo_count()`, `PIN()`, `FillRect()`, `DrawShapeAtXY()`, `DrawShape()`, `offset_rect()`
- **Notes:** Two modes: (1) energy weapons draw vertical fill bar; (2) bullet weapons draw grid of shapes, left-to-right or right-to-left per config; clamps ammo to max with PIN()

### update_inventory_panel
- **Signature:** `void HUD_Class::update_inventory_panel(bool force_redraw)`
- **Purpose:** Render inventory list with item names and counts; special handling for network statistics
- **Inputs:** `force_redraw` or checks `INVENTORY_IS_DIRTY(current_player)`
- **Outputs/Return:** None
- **Side effects:** Sets `ForceUpdate`; clears `INVENTORY_DIRTY_BIT`
- **Calls:** `get_interface_rectangle()`, `calculate_player_item_array()`, `get_header_name()`, `draw_inventory_header()`, `FillRect()`, `draw_inventory_time()`, `calculate_player_rankings()`, `get_player_data()`, `DrawText()`, `TextWidth()`, `draw_inventory_item()`
- **Notes:** Checks `DISABLE_NETWORKING` macro; displays game time countdown or kill limit in multiplayer; draws player rankings in network_statistics mode

### draw_inventory_item
- **Signature:** `void HUD_Class::draw_inventory_item(char *text, short count, short offset, bool erase_first, bool valid_in_this_environment)`
- **Purpose:** Render a single inventory row with item name and count
- **Inputs:** Item name text, count, row offset, erase flag, validity indicator
- **Outputs/Return:** None
- **Side effects:** Calls FillRect() and DrawText()
- **Calls:** `calculate_inventory_rectangle_from_offset()`, `FillRect()`, `DrawText()`, `sprintf()`
- **Notes:** Uses invalid_weapon_color if item not valid in current environment; always erases count digits, conditionally erases name area

## Control Flow Notes
HUD updates follow a per-frame dirty-flag pattern invoked by the main game loop:
1. **Frame entry:** `update_everything(time_elapsed)` called by engine each tick
2. **Conditional bypass:** If Lua texture palette is active, skip all normal HUD and render debug grid instead
3. **Selective redraw:** Each HUD subsystem (energy, oxygen, weapon, ammo, inventory) checks its dirty flag and `time_elapsed == NONE` (force)
4. **State reset:** All dirty flags cleared after redraw; `ForceUpdate` returned to signal any changes occurred
5. **Motion sensor:** Separate virtual method (`update_motion_sensor()`, `render_motion_sensor()`) for motion sensor updates (implementation in subclass)
6. **Network messages:** `draw_message_area()` called only in multiplayer (player_count > 1)

## External Dependencies
- **HUDRenderer.h** ΓÇö Base class `HUD_Class`, constants, structure definitions, virtual method interface
- **lua_script.h** ΓÇö Lua texture palette functions: `LuaTexturePaletteSize()`, `LuaTexturePaletteTexture()`, etc.
- **Defined elsewhere** ΓÇö `current_player`, `current_player_index`, `dynamic_world`, `interface_state`, `weapon_interface_definitions`, shape/texture IDs (_energy_bar, _weapon_panel_shape, etc.), UI helper functions (`get_interface_rectangle()`, `get_player_desired_weapon()`, `calculate_player_item_array()`, etc.), drawing primitives (`DrawShape`, `DrawText`, `FillRect`, `FrameRect` ΓÇö all virtual, subclass-implemented)
