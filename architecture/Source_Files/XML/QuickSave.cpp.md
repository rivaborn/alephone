# Source_Files/XML/QuickSave.cpp

## File Purpose
Implements a manager for auto-named saved games with metadata and preview images. Handles creation, loading, deletion, and UI dialogs for quick save management in the Aleph One game engine.

## Core Responsibilities
- Create, load, and delete quick save files
- Generate and cache map preview images for saves
- Provide dialog UI for selecting and managing saves
- Store and retrieve save metadata (level name, playtime, player count)
- Enumerate saves from the quick-saves directory
- Maintain LRU cache of preview images to limit memory usage

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `QuickSaveLoader` | class | Parses save files and enumerates directory; private implementation detail |
| `QuickSaveImageCache` | class | Singleton LRU cache for save preview images; limits to 100 items |
| `w_saves` | class | List widget displaying saves with previews and metadata |
| `w_save_name` | class | Text entry widget for renaming saves; closes on Return key |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `last_saved_game` | FileSpecifier | static | Tracks the most recently loaded save file |
| `last_saved_networked` | size_t | static | Records if last load was a networked (multiplayer) save |
| `RENDER_WIDTH`, `RENDER_HEIGHT` | int | const | Map preview render size (1280├ù720) |
| `PREVIEW_WIDTH`, `PREVIEW_HEIGHT` | int | const | Cached preview image size (128├ù72) |
| `RENDER_SCALE` | int | const | Overhead map scale for preview (from `OVERHEAD_MAP_MAXIMUM_SCALE`) |

## Key Functions / Methods

### create_quick_save
- Signature: `bool create_quick_save(void)`
- Purpose: Create a new quick save with current game state, metadata, and map preview
- Inputs: None (reads from `static_world`, `dynamic_world`, `local_player`)
- Outputs/Return: `bool` ΓÇô success/failure
- Side effects: Creates WAD file in quick-saves directory; may delete old unnamed saves if at limit
- Calls: `build_map_preview()`, `build_save_metadata()`, `save_game_file()`, `QuickSaves::instance()->delete_surplus_saves()`
- Notes: Uses timestamp as save file base name; formats time and tick count for display; enforces `maximum_quick_saves` preference

### load_quick_save_dialog
- Signature: `bool load_quick_save_dialog(FileSpecifier& saved_game)`
- Purpose: Show dialog to select a quick save to load; also provides rename/delete/export buttons
- Inputs: `saved_game` (FileSpecifier&) ΓÇô output parameter for selected file
- Outputs/Return: `bool` ΓÇô true if user selected a save or "Load Other"; sets `saved_game` and `last_saved_networked`
- Side effects: Clears and rebuilds save list; disables buttons if no selection; displays modal dialog
- Calls: `QuickSaves::instance()->enumerate()`, `load_quick_save_dialog()` creates sub-dialogs via callbacks
- Notes: Returns `LOAD_DIALOG_OTHER` (case 4) to load via file browser instead; enumerates saves at start and clears cache at end

### create_updated_save
- Signature: `void create_updated_save(QuickSave& save)`
- Purpose: Update an existing save file with new metadata (e.g., after rename)
- Inputs: `save` (QuickSave&) ΓÇô save with updated name; preserves existing game data and preview image
- Outputs/Return: void
- Side effects: Reads save WAD, extracts game data and preview image, rebuilds WAD with new metadata, replaces file
- Calls: `build_save_metadata()`, `build_meta_game_wad()`, WAD I/O functions
- Notes: Preserves the original game wad data; shows error alert if WAD write fails

### build_map_preview
- Signature: `static bool build_map_preview(std::ostringstream& ostream)`
- Purpose: Render overhead map to PNG/BMP image and write to stream
- Inputs: `ostream` (std::ostringstream&) ΓÇô output stream for image data
- Outputs/Return: `bool` ΓÇô success if render and save succeed
- Side effects: Temporarily modifies drawing port and `OGL_MapActive` flag; creates SDL surface
- Calls: `_set_port_to_custom()`, `_render_overhead_map()`, `_restore_port()`, `IMG_SavePNG_RW()` or `SDL_SaveBMP_RW()`
- Notes: Uses 1280├ù720 render size, scaled down to 128├ù72 cached; renders from `local_player->location`; fills black background first

### w_saves::draw_item
- Signature: `void w_saves::draw_item(QuickSaves::iterator it, SDL_Surface* s, int16 x, int16 y, uint16 width, bool selected) const`
- Purpose: Render a single save entry with preview image and metadata text
- Inputs: `it` (iterator), `s` (surface), `x`/`y` (position), `width`, `selected` (bool)
- Outputs/Return: void (draws to surface)
- Side effects: Blits preview image; draws text labels (name, formatted time, level, playtime)
- Calls: `QuickSaveImageCache::instance()->get()`, `draw_text()`, `utf8_to_mac_roman()`
- Notes: Uses theme colors for selected/default state; truncates text with clip rectangle

### QuickSaveImageCache::get
- Signature: `SDL_Surface* QuickSaveImageCache::get(std::string image_name)`
- Purpose: Retrieve a cached save preview image; load from WAD if not cached
- Inputs: `image_name` (std::string) ΓÇô base filename without extension
- Outputs/Return: SDL_Surface\* (may be null if load fails)
- Side effects: Moves item to front of LRU list; evicts oldest item if cache exceeds 100 items
- Calls: `WadImageCache::instance()->get_image()`
- Notes: Expects image as `<image_name>.sgaA` with tag `SAVE_IMG_TAG` at index `SAVE_GAME_METADATA_INDEX`

### QuickSaveLoader::ParseQuickSave
- Signature: `bool QuickSaveLoader::ParseQuickSave(FileSpecifier& file_name)`
- Purpose: Parse a single save file and extract metadata
- Inputs: `file_name` (FileSpecifier&) ΓÇô path to `.sgaA` save file
- Outputs/Return: `bool` ΓÇô true if header read successfully (even if metadata missing)
- Side effects: Adds parsed `QuickSave` to `QuickSaves::instance()`
- Calls: WAD read functions, `InfoTree::load_ini()`, `QuickSaves::instance()->add()`
- Notes: Catches and logs `InfoTree::ini_error`; returns false only on file open failure

### dialog_rename, dialog_delete, dialog_export
- Signature: `static void dialog_rename(void *arg)` (and similar)
- Purpose: Modal dialog callbacks for rename/delete/export operations
- Inputs: `arg` (void\*) ΓÇô pointer to parent dialog
- Outputs/Return: void
- Side effects: Modify save metadata or file system; update parent dialog state if needed
- Calls: `dialog_rename` ΓåÆ `create_updated_save()`, `saves_w->update_selected()`;
        `dialog_delete` ΓåÆ `delete_quick_save()`, disables buttons if no saves left;
        `dialog_export` ΓåÆ `FileSpecifier::WriteDialog()`, `CopyContents()`

## Control Flow Notes

**Save Creation Path:** `create_quick_save()` ΓåÆ `build_map_preview()` + `build_save_metadata()` ΓåÆ `save_game_file()` ΓåÆ `delete_surplus_saves()` (if at limit)

**Load Path:** `load_quick_save_dialog()` ΓåÆ `QuickSaves::enumerate()` ΓåÆ `QuickSaveLoader::ParseDirectory()` / `ParseQuickSave()` ΓåÆ Display `w_saves` widget ΓåÆ Return selected `FileSpecifier` to caller

**Update Path:** Rename/Delete/Export dialogs ΓåÆ `create_updated_save()` (rename) or `delete_quick_save()` (delete) ΓåÆ Update widget state

**Image Caching:** First draw of save item ΓåÆ `QuickSaveImageCache::get()` ΓåÆ Check cache ΓåÆ Load via `WadImageCache` if miss ΓåÆ Cache with LRU eviction

Not inferable: Connection to actual game load/save (game_wad.h functions are external; this module only manages metadata/UI).

## External Dependencies

- **FileHandler.h** ΓÇô `FileSpecifier`, `OpenedFile`, directory operations
- **world.h** ΓÇô `world_point2d`, `world_point3d`, coordinate types
- **map.h** ΓÇô `TICKS_PER_MINUTE`, `TICKS_PER_SECOND`, `local_player`, `dynamic_world`, `static_world`
- **wad.h** ΓÇô WAD file I/O (`read_wad_header()`, `read_indexed_wad_from_file()`, `write_wad()`, etc.)
- **overhead_map.h** ΓÇô `_render_overhead_map()`, `overhead_map_data`, rendering modes
- **screen_drawing.h** ΓÇô `_set_port_to_custom()`, `_restore_port()`, `draw_text()`
- **sdl_dialogs.h, sdl_widgets.h** ΓÇô Dialog and widget base classes
- **WadImageCache.h** ΓÇô `WadImageCache::instance()`, `WadImageDescriptor`
- **InfoTree.h** ΓÇô `InfoTree` for metadata parsing/generation
- **SDL2/SDL_image.h** ΓÇô `IMG_SavePNG_RW()` (conditional)
- **boost/algorithm/string/** ΓÇô String manipulation (replace, ends_with)
- Defined elsewhere: `OGL_MapActive` (flag), `environment_preferences->maximum_quick_saves`, various `_typecode_*` and tag constants
