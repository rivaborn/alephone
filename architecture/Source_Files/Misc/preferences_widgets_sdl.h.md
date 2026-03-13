# Source_Files/Misc/preferences_widgets_sdl.h

## File Purpose
This header defines SDL-based preference dialog widgets for file/directory selection, crosshair display, and plugin listing in the Aleph One game engine. It provides components for the preferences UI, including file choosers with validation and plugin list display.

## Core Responsibilities
- File and directory selection UI components with path management
- Environment item representation with hierarchy (indentation) and selectability states
- Crosshair preview display widget
- Plugin list rendering with dual-line item layout
- File selection callback mechanism and modern binding interface
- Theme-aware list rendering with indentation and selection highlighting

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| env_item | struct | Represents a file/directory entry with path, name, indentation level, and selectability flag |
| w_env_list | class (extends w_list<env_item>) | List widget displaying environment items with custom theme-aware rendering |
| w_env_select | class (extends w_select_button) | Selection button for file/directory selection with path validation and callbacks |
| EnvSelectWidget | class (extends SDLWidgetWidget, Bindable<FileSpecifier>) | Modern wrapper providing callback and binding support for file selection |
| w_crosshair_display | class (extends widget) | Non-interactive display widget for crosshair preview rendering |
| w_plugins | class (extends w_list_base) | Specialized list widget for plugin display with dual-line item rendering |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| data_search_path | extern vector<DirectorySpecifier> | global | Search paths for locating data files (defined in shell_sdl.cpp) |
| sFileChooserInvalidFileString | extern const char* const | global | String displayed when a file path is invalid or empty |

## Key Functions / Methods

### w_env_select::set_path
- Signature: `void set_path(const char *p)`
- Purpose: Set the current file path and update the displayed item name
- Inputs: `p` ΓÇö file path string (may be empty or non-existent)
- Outputs/Return: None; modifies button display
- Side effects: Updates `item` (FileSpecifier), `item_name` buffer; calls `set_selection()` to refresh button text
- Calls: `FileSpecifier::Exists()`, `FileSpecifier::GetName()`, `FileSpecifier::HideExtension()`, `set_selection()`
- Notes: Wraps missing files as "[?filename]"; empty paths show invalid file string

### w_env_list::draw_item
- Signature: `void draw_item(vector<env_item>::const_iterator i, SDL_Surface *s, int16 x, int16 y, uint16 width, bool selected) const`
- Purpose: Render a single environment item to the list display
- Inputs: `i` ΓÇö item iterator; `s` ΓÇö target surface; `x`, `y` ΓÇö position; `width` ΓÇö available width; `selected` ΓÇö highlight flag
- Outputs/Return: None; draws to SDL_Surface
- Side effects: Renders text with theme colors; sets clip rectangle for safe drawing
- Calls: `font->get_ascent()`, `get_theme_color()`, `set_drawing_clip_rectangle()`, `draw_text()`, `FileSpecifier::HideExtension()`
- Notes: Color depends on selectability (ITEM_WIDGET vs LABEL_WIDGET theme); applies `indent * 8` pixel offset

### w_env_list::item_selected
- Signature: `void item_selected(void)`
- Purpose: Handle user selection of an environment item
- Inputs: None
- Outputs/Return: None
- Side effects: Closes parent dialog with exit code 0
- Calls: `parent->quit(0)`
- Notes: Simple callback; selection is already set by base class w_list_base before this fires

### w_plugins::draw_items
- Signature: `void draw_items(SDL_Surface* s) const`
- Purpose: Render all visible plugin items in the list
- Inputs: `s` ΓÇö destination surface
- Outputs/Return: None; draws to surface
- Side effects: Iterates visible range [top_item, top_item + shown_items)
- Calls: `draw_item()` (private), `item_height()`
- Notes: Item height is `2 * font->get_line_height() + font->get_line_height() / 2 + 2` for dual-line layout

## Control Flow Notes
**File Selection**: Preferences dialog instantiates `w_env_select` button. User clicks ΓåÆ opens file picker dialog with `w_env_list` ΓåÆ `item_selected()` fires ΓåÆ parent dialog closes with selection. For modern code, `EnvSelectWidget` wraps this with callback support.

**Display**: Crosshair display and plugin lists are rendered on-demand within dialogs; marked dirty to force redraw each frame.

## External Dependencies
- **sdl_widgets.h** ΓÇö Base widget classes: `w_list<T>`, `w_list_base`, `w_select_button`, `widget`, `SDLWidgetWidget`, `Bindable<T>`
- **sdl_fonts.h** ΓÇö Font metrics/rendering: `font_info`
- **screen_drawing.h** ΓÇö Drawing primitives: `set_drawing_clip_rectangle()`, `draw_text()`
- **find_files.h** ΓÇö File utilities: `DirectorySpecifier`, `FileSpecifier`
- **interface.h** ΓÇö UI constants/structures
- **Plugins.h** ΓÇö Plugin structure definitions (defined elsewhere)
