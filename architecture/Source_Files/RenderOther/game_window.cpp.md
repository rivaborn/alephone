# Source_Files/RenderOther/game_window.cpp

## File Purpose
Manages the game's heads-up display (HUD) and interface state for the Aleph One engine. Handles HUD buffering, weapon/inventory display configuration, dynamic interface updates, and runtime customization through MML (Marathon Markup Language) parsing.

## Core Responsibilities
- Initialize and update the HUD each frame (software-rendered path)
- Maintain interface state: motion sensor, weapon/ammo, shields, oxygen, inventory
- Define and manage 10 weapon interface layouts with ammunition display parameters
- Buffer HUD to an SDL surface for efficient rendering
- Handle inventory screen scrolling and switching
- Parse and apply MML-based interface customizations at runtime
- Mark interface elements dirty to trigger selective redraws
- Manage motion sensor initialization and state

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `interface_state_data` | struct | Holds current UI state (motion sensor, weapon/ammo/shield/oxygen flags) |
| `weapon_interface_data` | struct | Defines visual layout for one weapon: panel position, ammo slot positions, multi-weapon alt shape |
| `weapon_interface_ammo_data` | struct | Describes one ammo display slot: type, screen coords, grid layout, bullet/empty sprites |
| `HUD_SW_Class` | class | Software HUD renderer; inherits from `HUD_Class` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MotionSensorActive` | bool | global | Whether the motion sensor blip display is enabled |
| `interface_state` | `interface_state_data` | global | Current HUD state (dirty flags, etc.) |
| `weapon_interface_definitions` | `weapon_interface_data[10]` | global | Layout config for all 10 weapons; initialized with hardcoded defaults |
| `HUD_SW` | `HUD_SW_Class` | global | Software HUD renderer instance |
| `HUD_Buffer` | `SDL_Surface*` | global (from screen_sdl.cpp) | Offscreen surface for buffered HUD rendering |
| `static_hud_pict` | `std::shared_ptr<SDL_Surface>` | static (in `draw_panels()`) | Cached static HUD background image |
| `hud_pict_not_found` | bool | static (in `draw_panels()`) | Flag indicating whether background image load failed |
| `original_weapon_interface_definitions` | `weapon_interface_data*` | global | Backup of defaults before MML overrides; freed on reset |
| `original_menu_item_order` | `std::vector<int>` | global | Backup of menu order before MML overrides |

## Key Functions / Methods

### initialize_game_window
- **Signature:** `void initialize_game_window(void)`
- **Purpose:** Set up the motion sensor at game start.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects (global state, I/O, alloc):** Calls `initialize_motion_sensor()` with hardcoded collection and shape descriptors.
- **Calls (direct calls visible in this file):** `initialize_motion_sensor()`
- **Notes:** Part of startup sequence; motion sensor visuals are configured by the 5 passed descriptors (mount, virgin mount, alien, friend, enemy sprites).

### draw_interface
- **Signature:** `void draw_interface(void)`
- **Purpose:** Render the entire interface frame (panels) each frame.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects (global state, I/O, alloc):** Skips rendering if OpenGL mode is active. Calls `draw_panels()` and `validate_world_window()`.
- **Calls (direct calls visible in this file):** `draw_panels()`, `validate_world_window()`, `alephone::Screen::instance()->openGL()`
- **Notes:** Software-rendering path only; no-op if OpenGL is enabled.

### update_interface
- **Signature:** `void update_interface(short time_elapsed)`
- **Purpose:** Update HUD elements each frame; redraw only changed parts (or all if `time_elapsed == NONE`).
- **Inputs:** `time_elapsed` ΓÇô elapsed ticks since last update; `NONE` means full redraw.
- **Outputs/Return:** None.
- **Side effects (global state, I/O, alloc):** Resets motion sensor, ensures HUD buffer exists, calls `HUD_SW.update_everything()`, may request HUD drawing.
- **Calls (direct calls visible in this file):** `reset_motion_sensor()`, `alephone::Screen::instance()`, `ensure_HUD_buffer()`, `_set_port_to_HUD()`, `HUD_SW.update_everything()`, `_restore_port()`, `RequestDrawingHUD()`
- **Notes:** Software-rendering path only; skipped if OpenGL or Lua HUD enabled.

### mark_*_display_as_dirty
- **Signature:** `void mark_weapon/ammo/shield/oxygen_display_as_dirty(void)`
- **Purpose:** Flag individual HUD elements for redraw.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Sets corresponding dirty flag in `interface_state`.
- **Notes:** Trivial accessors; called whenever game state affecting that display changes.

### mark_player_inventory_as_dirty / mark_player_inventory_screen_as_dirty
- **Signature:** `void mark_player_inventory_as_dirty(short player_index, short dirty_item)` / `void mark_player_inventory_screen_as_dirty(short player_index, short screen)`
- **Purpose:** Mark player inventory dirty; optionally switch to a specific item category or screen.
- **Inputs:** `player_index`, optional `dirty_item` (item kind) or `screen` (inventory screen type).
- **Outputs/Return:** None.
- **Side effects:** May call `set_current_inventory_screen()` if item kind differs from current screen.
- **Calls:** `get_player_data()`, `get_item_kind()`, `set_current_inventory_screen()`, macros `GET_CURRENT_INVENTORY_SCREEN`, `SET_INVENTORY_DIRTY_STATE`
- **Notes:** Auto-switches inventory view if new item type is picked up (except powerups).

### scroll_inventory
- **Signature:** `void scroll_inventory(short dy)`
- **Purpose:** Cycle through inventory screens (up or down) to the next screen with items.
- **Inputs:** `dy` ΓÇô direction: > 0 means forward, Γëñ 0 means backward.
- **Outputs/Return:** None.
- **Side effects:** Calls `set_current_inventory_screen()`, sets inventory dirty flag.
- **Calls:** `GET_CURRENT_INVENTORY_SCREEN()`, `calculate_player_item_array()`, `set_current_inventory_screen()`, `SET_INVENTORY_DIRTY_STATE()`
- **Notes:** Wraps around (modulo); in multiplayer, includes network-statistics screen as an option. Uses asserts to validate screen range.

### set_current_inventory_screen
- **Signature:** `static void set_current_inventory_screen(short player_index, short screen)`
- **Purpose:** Update the player's current inventory screen and set display timeout.
- **Inputs:** `player_index`, `screen` (0ΓÇô6).
- **Outputs/Return:** None.
- **Side effects:** Modifies `player->interface_flags` and `player->interface_decay`.
- **Calls:** `get_player_data()`
- **Notes:** Asserts screen is in range [0, 7). Uses bitmask to pack screen into interface flags.

### ensure_HUD_buffer
- **Signature:** `void ensure_HUD_buffer(void)`
- **Purpose:** Allocate the HUD surface on first use.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Creates 640├ù480 32-bit RGBA SDL surface if not already allocated.
- **Calls:** `SDL_CreateRGBSurface()`, `alert_out_of_memory()`
- **Notes:** Called before any HUD drawing. 32-bit RGBA with explicit channel masks (red=0x00ff0000, green=0x0000ff00, blue=0x000000ff, alpha=0xff000000).

### draw_panels
- **Signature:** `void draw_panels(void)`
- **Purpose:** Draw the static HUD background and dynamic elements to the HUD surface.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Loads and caches static HUD background image; blits to HUD_Buffer; calls `HUD_SW.update_everything(NONE)` for dynamic elements; requests HUD drawing.
- **Calls:** `ensure_HUD_buffer()`, `get_picture_resource_from_images()`, `picture_to_surface()`, `LuaTexturePaletteSize()`, `SDL_BlitSurface()`, `SDL_FillRect()`, `_set_port_to_HUD()`, `HUD_SW.update_everything()`, `_restore_port()`, `RequestDrawingHUD()`
- **Notes:** Skips if OpenGL enabled. Handles Lua texture palette specially (fills rect instead of blitting image).

### reset_mml_interface
- **Signature:** `void reset_mml_interface(void)`
- **Purpose:** Restore weapon interface and menu item order to defaults.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Restores `weapon_interface_definitions[]` and `menu_item_order` from backups; frees backup memory.
- **Calls:** `free()`
- **Notes:** Called when discarding MML changes. Asserts backups exist before restoring.

### parse_mml_interface
- **Signature:** `void parse_mml_interface(const InfoTree& root)`
- **Purpose:** Parse MML XML and apply customizations to interface (motion sensor, rectangles, menu items, colors, fonts, weapon layouts).
- **Inputs:** `root` ΓÇô root InfoTree node of interface section.
- **Outputs/Return:** None.
- **Side effects:** Backs up defaults on first call; modifies `weapon_interface_definitions[]`, `menu_item_order`, interface colors/fonts, motion sensor active flag, vidmaster settings.
- **Calls:** `read_attr()`, `read_indexed()`, `read_color()`, `read_font()`, `get_interface_rectangle()`, `get_interface_color()`, `get_interface_font()`, `malloc()`, `std::copy()`, helpers on `root` and child `InfoTree` nodes.
- **Notes:** Supports child elements: `<rect>`, `<menu_item>`, `<color>`, `<font>`, `<vidmaster>`, `<weapon>` (with nested `<ammo>`). Uses indexed lookups with bounds checking.

## Control Flow Notes
- **Startup:** `initialize_game_window()` called once.
- **Each frame:** 
  - `draw_interface()` renders static frame (if fullscreen disabled and not OpenGL).
  - `update_interface(time_elapsed)` updates dynamic HUD and buffers to surface.
  - Dirty flags control selective redraw.
- **MML load:** `parse_mml_interface()` called during game initialization or preference reload; backs up defaults and applies overrides.
- **Reset:** `reset_mml_interface()` restores defaults; called when exiting MML-customized state.
- **Inventory:** `scroll_inventory()` and mark functions manage the inventory UI state machine.

## External Dependencies
- **SDL2:** `SDL_Surface`, `SDL_CreateRGBSurface`, `SDL_BlitSurface`, `SDL_FillRect`, `SDL_Rect`, `SDL_FreeSurface`
- **HUDRenderer_SW.h:** `HUD_SW_Class`, software HUD rendering
- **screen.h:** `alephone::Screen::instance()`, `RequestDrawingHUD()`, `validate_world_window()`, `game_window_is_full_screen()` (defined elsewhere)
- **shell.h:** `draw_panels()` (extern), `vidmasterStringSetID`, `vidmasterLevelOffset` (defined in shell.cpp)
- **preferences.h:** `GET_GAME_OPTIONS()` macro
- **InfoTree.h:** `InfoTree` class for XML parsing
- **FontHandler.h, screen_definitions.h, images.h, interface_menus.h:** Color, font, and shape definitions
- **Undeclared externs:** `_set_port_to_HUD()`, `_restore_port()`, `initialize_motion_sensor()`, `reset_motion_sensor()`, `current_player_index`, `current_player`, `dynamic_world`, `get_player_data()`, `get_item_kind()`, `calculate_player_item_array()`, `get_interface_rectangle()`, `get_interface_color()`, `get_interface_font()`, `alert_out_of_memory()`, `LuaTexturePaletteSize()`, `get_picture_resource_from_images()`, `picture_to_surface()` (all defined elsewhere in engine).
