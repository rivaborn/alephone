# Source_Files/Files/FileHandler.cpp

## File Purpose
Cross-platform file I/O abstraction and dialog system for the Aleph One game engine. Wraps SDL_RWops and boost.filesystem to provide path manipulation, file operations, resource fork handling, and native/custom file selection dialogs with ZIP archive support.

## Core Responsibilities
- Manage opened files, resources, and resource forks via SDL_RWops abstractions
- Transparently handle macOS compatibility formats (AppleSingle, MacBinary) on read
- Detect file types by extension or content inspection (magic numbers)
- Provide FileSpecifier class for path construction, validation, and file operations
- Support ZIP archive enumeration (ZZIP integration)
- Implement file selection dialogs (native NFD or custom SDL-based fallback)
- Maintain per-user directory specifications (data, preferences, saves, cache, recordings)
- Provide directory browsing widget with sorting and navigation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| OpenedFile | class | Wraps SDL_RWops; tracks fork offsets for Mac formats |
| opened_file_device | class | Boost.iostreams device adapter for OpenedFile |
| LoadedResource | class | Manages malloc'd resource data; auto-unload on destroy |
| OpenedResourceFile | class | Resource fork abstraction using global res file state |
| FileSpecifier | class | Path representation; handles file I/O, type detection, dialogs |
| dir_entry | struct | Directory listing: name, is_directory, modification date |
| extension_mapping | struct | File extension ΓåÆ typecode mapping with case sensitivity flag |
| w_directory_browsing_list | class | SDL widget for directory navigation and file selection |
| FileDialog | class | Base for read/write dialogs; manages layout and callbacks |
| ReadFileDialog | class | File open dialog with type-specific directory defaults |
| WriteFileDialog | class | File save dialog with overwrite confirmation |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| data_search_path | vector\<DirectorySpecifier\> | extern | Search paths for locating data files |
| local_data_dir, preferences_dir, saved_games_dir, quick_saves_dir, image_cache_dir, recordings_dir | DirectorySpecifier | extern | Per-user directories for different file categories |
| alephone_extensions[] | const char*[] | static | Game-specific extensions (.sceA, .sgaA, .filA, etc.) for UI display |
| extensions[] | extension_mapping[] | static | Mapping table: file extensions to typecodes with case rules |
| sort_by_labels[] | const char*[] | static | UI labels for sort dropdown ("name", "date") |
| typecode_filters | std::map\<Typecode, vector\<nfdu8filteritem_t\>\> | static | Native dialog file filters by type |

## Key Functions / Methods

### FileSpecifier::Open (file reading)
- **Signature**: `bool FileSpecifier::Open(OpenedFile& OFile, bool Writable)`
- **Purpose**: Open file for reading or writing; transparently handle AppleSingle/MacBinary on read
- **Inputs**: Reference to OpenedFile, Writable flag
- **Outputs/Return**: True if file opened successfully
- **Side effects**: Populates OFile.f (SDL_RWops), OFile.is_forked, OFile.fork_offset, OFile.fork_length; seeks to correct offset
- **Calls**: SDL_RWFromZZIP, SDL_RWFromFile, is_applesingle, is_macbinary, SDL_RWseek
- **Notes**: Checks for ZZIP support; sets err on failure; automatically detects and strips Mac format headers

### FileSpecifier::GetType
- **Signature**: `Typecode FileSpecifier::GetType()`
- **Purpose**: Determine file type by extension or by reading and parsing file content (magic numbers)
- **Inputs**: File path in this.name
- **Outputs/Return**: Typecode enum (e.g., _typecode_scenario, _typecode_shapes, _typecode_sounds)
- **Side effects**: Opens and closes file; sets err; reads up to 128+ bytes
- **Calls**: Open, SDL_ReadBE32, SDL_ReadBE16, SetPosition, GetLength
- **Notes**: Fast path via extension matching; falls back to content inspection for maps, shapes, sounds; returns _typecode_unknown if indeterminate

### FileSpecifier::ReadDirectory
- **Signature**: `bool FileSpecifier::ReadDirectory(vector<dir_entry>& vec)`
- **Purpose**: List directory contents; exclude dot-prefixed regular files and special types
- **Inputs**: Reference to output vector
- **Outputs/Return**: True if successful; vec populated with sorted dir_entry items
- **Side effects**: Clears vec; sets err field; calls filesystem stat
- **Calls**: fs::directory_iterator, fs::last_write_time, path_to_utf8
- **Notes**: Sorts directories before files; skips symlinks and special files; follows symlink targets

### FileSpecifier::ReadDialog
- **Signature**: `bool FileSpecifier::ReadDialog(Typecode type, const char* prompt)`
- **Purpose**: Show file selection dialog (native or custom); return selected path
- **Inputs**: Typecode (for directory defaults), optional prompt override
- **Outputs/Return**: True if user selected file; path stored in this.name
- **Side effects**: May toggle fullscreen; calls NFD or custom dialog; modifies this.name on success
- **Calls**: NFD_OpenDialogU8_With, ReadFileDialog::Run, toggle_fullscreen, MainScreenWindow
- **Notes**: Sets start directory based on file type; uses native dialogs if HAVE_NFD and environment_preferences->use_native_file_dialogs

### FileSpecifier::WriteDialog
- **Signature**: `bool FileSpecifier::WriteDialog(Typecode type, const char* prompt, const char* default_name)`
- **Purpose**: Show save dialog with filename entry and overwrite confirmation
- **Inputs**: Typecode, optional prompt, optional default filename
- **Outputs/Return**: True if user confirmed save with valid filename
- **Side effects**: Loops until valid filename; may toggle fullscreen; calls confirm dialog
- **Calls**: NFD_SaveDialogU8_With, WriteFileDialog::Run, confirm_save_choice, play_dialog_sound
- **Notes**: Adds extension if configured; strips extension from default_name if already present; handles native and custom dialogs

### FileSpecifier::SetNameWithPath
- **Signature**: `bool FileSpecifier::SetNameWithPath(const char* NameWithPath)` and `bool FileSpecifier::SetNameWithPath(const char* NameWithPath, const DirectorySpecifier& Directory)`
- **Purpose**: Locate file in search path or relative to given directory
- **Inputs**: Relative path (Unix-style forward slashes), optional starting directory
- **Outputs/Return**: True if file found; this.name set to full path
- **Side effects**: Sets err to ENOENT if not found
- **Calls**: local_path_separators, Exists
- **Notes**: Iterates through data_search_path; converts platform separators; returns on first match

### w_directory_browsing_list::item_selected
- **Signature**: `void w_directory_browsing_list::item_selected()`
- **Purpose**: Handle user selection of directory entry in list widget
- **Inputs**: None (uses inherited selection index)
- **Outputs/Return**: None
- **Side effects**: Navigates into directory or invokes file_selected callback
- **Calls**: refresh_entries, announce_directory_changed, file_selected
- **Notes**: Called by w_list base class; entry must be directory to navigate; regular files trigger callback

### FileSpecifier::CopyContents
- **Signature**: `bool FileSpecifier::CopyContents(FileSpecifier& source_name)`
- **Purpose**: Copy file contents from source to destination (this)
- **Inputs**: Source FileSpecifier
- **Outputs/Return**: True if copy succeeded
- **Side effects**: Deletes destination file; sets err on failure; opens source and destination files
- **Calls**: Delete, Open, GetLength, Read, Write
- **Notes**: Uses 1 KB buffer; aborts and deletes destination on I/O error; preserves error codes

### FileSpecifier::ReadZIP
- **Signature**: `bool FileSpecifier::ReadZIP(vector<string>& vec)`
- **Purpose**: Enumerate file entries within a ZIP archive
- **Inputs**: Reference to output vector
- **Outputs/Return**: True if ZIP opened; vec populated with entry names
- **Side effects**: Clears vec; sets err to ENOTSUP if HAVE_ZZIP not defined
- **Calls**: zzip_dir_open_ext_io, zzip_dir_read, zzip_dir_close
- **Notes**: Only functional if HAVE_ZZIP; uses UTF-8 compatible zzip I/O handlers

## Control Flow Notes

This file is **not** part of the core game loop (init/frame/update/render/shutdown). Instead:
- **Initialization phase**: Directory specifications (data_search_path, local_data_dir, etc.) are populated by shell_sdl.cpp during app startup
- **On-demand invocation**: File I/O and dialogs occur when game menus call ReadDialog/WriteDialog or when code explicitly opens files
- **Dialog-driven flow**: ReadFileDialog and WriteFileDialog are modal; they block until user action or cancellation, then update this.name
- **Filesystem integration**: FileSpecifier operations (Exists, GetType, ReadDirectory) are called synchronously during file system queries

The dialog classes integrate with the SDL widget framework (w_list, w_button, w_select) and render via screen.h drawing functions.

## External Dependencies

**Notable includes**:
- SDL2 (SDL_RWops, SDL_RWseek, SDL endian macros)
- boost::filesystem (directory_iterator, path, is_directory, last_write_time, remove, rename, create_directory)
- boost::iostreams (seekable_device_tag, stream_offset)
- boost::algorithm (ends_with)
- ZZIP (zzip_dir_open_ext_io, zzip_dir_read; optional)
- NFD (native file dialogs; optional)

**External symbols (defined elsewhere)**:
- Global directories: data_search_path, local_data_dir, preferences_dir, saved_games_dir, quick_saves_dir, image_cache_dir, recordings_dir (from shell_sdl.cpp)
- Mac format detection: is_applesingle, is_macbinary
- UI framework: dialog, w_list, w_button, w_select, w_title, w_static_text, w_text_entry, vertical_placer, horizontal_placer (sdl_dialogs.h, sdl_widgets.h)
- Drawing: draw_text, get_theme_color, font, set_drawing_clip_rectangle (sdl_widgets.h)
- Screen: get_screen_mode, toggle_fullscreen, MainScreenWindow, update_game_window (screen.h, interface.h)
- Preferences: environment_preferences (global settings object)
- Audio: play_dialog_sound (SoundManager.h)
