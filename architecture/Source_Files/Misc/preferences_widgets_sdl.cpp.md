# Source_Files/Misc/preferences_widgets_sdl.cpp

## File Purpose
Implements SDL-specific preferences dialog widgets for Aleph One game engine. Provides widgets for environment file selection (maps, physics, shapes, sounds), crosshair preview rendering, and plugin list management, with optional Steam Workshop integration.

## Core Responsibilities
- Construct and manage environment file selection dialogs with directory browsing
- Integrate Steam Workshop item discovery and filtering for downloadable content
- Render crosshair appearance preview in preferences dialog
- Display and manage plugin list with enable/disable toggling
- Handle file path validation and callback notification on selection changes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `env_item` | struct | Wraps a file specifier with display name, indentation level, and selectability flag |
| `w_env_list` | class | List widget for displaying environment files with hierarchical indentation by directory |
| `w_env_select` | class | Button widget that opens a file selection dialog and stores chosen file path |
| `EnvSelectWidget` | class | Data-binding wrapper around `w_env_select` for integrating with UI framework |
| `w_crosshair_display` | class | Widget that renders a preview of the current crosshair appearance |
| `w_plugins` | class | List widget for displaying plugins with enabled/disabled/compatible status |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `use_lua_hud_crosshairs` | bool | extern | Flag controlling whether Lua HUD overrides crosshair rendering |
| `data_search_path` | `vector<DirectorySpecifier>` | extern | Directories to search for environment files |
| `subscribed_workshop_items` | `vector<item>` | extern (Steam only) | Steam Workshop items currently subscribed by player |

## Key Functions / Methods

### w_env_select::select_item
- **Signature:** `void select_item(dialog *parent)`
- **Purpose:** Build and display modal dialog for selecting an environment file (map/physics/shapes/sounds/theme).
- **Inputs:** `parent` dialog context.
- **Outputs/Return:** None; updates internal `item` path and triggers callback.
- **Side effects:** Creates temporary dialog, clears screen, reads from `data_search_path` and Steam Workshop items, calls `mCallback` on success.
- **Calls:** `add_workshop_items`, `FindAllFiles::Find`, `FileSpecifier::SplitPath`, dialog construction methods, `set_path`, callback invocation.
- **Notes:** Groups files by directory with non-selectable folder headers. Handles "LOAD OTHER" file chooser on non-Mac App Store builds. Validates file existence and marks missing files with `[?ΓÇª]` prefix.

### add_workshop_items (overloaded variant 1)
- **Signature:** `static void add_workshop_items(vector<env_item>&, Typecode, ItemType, std::optional<ContentType>, const string&)`
- **Purpose:** Filter and add Steam Workshop items of a specific type to environment selection list.
- **Inputs:** `items` (output vector), `type` (typecode), `item_type` (workshop category), `content_type` (optional Solo/Net filter), `header` (section title).
- **Outputs/Return:** Appends to `items` vector.
- **Side effects:** Reads `subscribed_workshop_items`, creates `FindAllFiles` context per item, sorts results case-insensitively.
- **Calls:** `FindAllFiles::Find`, `FileSpecifier::SplitPath`, `std::sort`, `std::lexicographical_compare`.
- **Notes:** Returns early if no files found. Adds header only if items exist.

### add_workshop_items (overloaded variant 2)
- **Signature:** `static void add_workshop_items(vector<env_item>&, Typecode, bool prefer_net)`
- **Purpose:** Route workshop items by content type (Solo vs. Net) based on preference, supporting scenarios and scripts.
- **Inputs:** `items` (output), `type` (typecode), `prefer_net` (whether to prioritize net-type content).
- **Outputs/Return:** Calls variant 1 multiple times with appropriate filters.
- **Side effects:** Dispatches to variant 1; order of calls determines display order.
- **Calls:** Variant 1 of `add_workshop_items`.
- **Notes:** Special handling for scenarios and scripts (prefer_net reorders sections); physics, sounds, shapes use single unfiltered section.

### w_crosshair_display::draw
- **Signature:** `void draw(SDL_Surface *s) const`
- **Purpose:** Render crosshair appearance preview to destination SDL surface.
- **Inputs:** `s` (destination SDL surface).
- **Outputs/Return:** None; draws to surface.
- **Side effects:** Modifies `surface` member; temporarily disables Lua HUD crosshairs and toggles `Crosshairs_IsActive` state.
- **Calls:** `SDL_FillRect`, `draw_rectangle`, `Crosshairs_SetActive`, `Crosshairs_Render`, `SDL_BlitSurface`.
- **Notes:** Preserves and restores prior `use_lua_hud_crosshairs` flag and crosshair active state after rendering.

### w_plugins::item_selected
- **Signature:** `void item_selected()`
- **Purpose:** Toggle enabled state of selected plugin and mark widget dirty for redraw.
- **Inputs:** None (uses `get_selection()` internally).
- **Outputs/Return:** None.
- **Side effects:** Flips `plugin.enabled` flag, sets `dirty = true`, triggers `get_owning_dialog()->draw_dirty_widgets()`.
- **Calls:** `get_selection()`, dialog dirty-flag methods.

### w_plugins::draw_items
- **Signature:** `void draw_items(SDL_Surface* s) const`
- **Purpose:** Render visible range of plugin list items to SDL surface.
- **Inputs:** `s` (destination surface).
- **Outputs/Return:** None; draws to surface.
- **Side effects:** Iterates from `top_item` and calls `draw_item` for each visible row.
- **Calls:** `draw_item` (for each item).
- **Notes:** Respects `top_item`, `shown_items`, and `num_items` to render only visible portion.

### w_plugins::draw_item
- **Signature:** `void draw_item(Plugins::iterator it, SDL_Surface* s, int16 x, int16 y, uint16 width, bool selected) const`
- **Purpose:** Render single plugin entry with name, version, status, type tags, and description.
- **Inputs:** Plugin iterator, surface, position, width, selection state.
- **Outputs/Return:** None; draws to surface.
- **Side effects:** Sets drawing clip rectangles and calls `draw_text` multiple times.
- **Calls:** `set_drawing_clip_rectangle`, `get_theme_color`, `text_width`, `draw_text`.
- **Notes:** Renders three lines: (1) name + version + status, (2) type tags (italic), (3) description. Status shown as "Incompatible", "Disallowed", "Enabled", or "Disabled" with color coding.

## Control Flow Notes
- **Init:** Constructors allocate SDL surface (`w_crosshair_display`) or store references (`w_env_select`, `w_plugins`).
- **Update/Frame:** `draw` methods called during dialog rendering pass; `item_selected` triggered on user interaction.
- **Shutdown:** Destructors free SDL resources.
- **Environment Dialog Flow:** User clicks env-select button ΓåÆ `select_item_callback` ΓåÆ `select_item` builds list ΓåÆ modal dialog runs ΓåÆ callback notified on selection.

## External Dependencies
- **Notable includes:** `cseries.h`, `find_files.h`, `collection_definition.h`, `sdl_widgets.h`, `screen.h`, `Plugins.h`, `preferences.h`, `Crosshairs.h`
- **Extern symbols:** `use_lua_hud_crosshairs`, `data_search_path`, `subscribed_workshop_items` (Steam), `sFileChooserInvalidFileString`
- **Key dependencies (defined elsewhere):** `FindAllFiles`, `FileSpecifier`, `dialog`, `w_list_base`, `Crosshairs_IsActive`, `Crosshairs_SetActive`, `Crosshairs_Render`, `get_theme_color`, theme/drawing utilities, `Plugin` struct
