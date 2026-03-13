# Source_Files/Files/wad_prefs.cpp

## File Purpose
Provides persistent storage and retrieval of application preferences using WAD (Where's All the Data?) files. Wraps WAD file I/O with initialization, validation, and corruption recovery logic.

## Core Responsibilities
- Open/create preferences files with fallback recovery on corruption
- Retrieve preference entries, auto-initializing and validating them
- Write modified preferences back to disk with atomic file recreation
- Handle version mismatches and system errors gracefully

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `preferences_info` | struct | Holds file path and WAD data pointer; manages one preferences file |
| `prefs_initializer` | typedef | Callback to initialize newly-allocated preference data |
| `prefs_validater` | typedef | Callback to validate/repair preference data; returns true if modified |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `prefInfo` | `preferences_info*` | global | Singleton pointer to current active preferences file |
| `CURRENT_PREF_WADFILE_VERSION` | macro (0) | file | WAD data version identifier |

## Key Functions / Methods

### w_open_preferences_file
- **Signature:** `bool w_open_preferences_file(char *PrefName, Typecode Type)`
- **Purpose:** Open or create a preferences file; initialize global `prefInfo`.
- **Inputs:** `PrefName` (filename), `Type` (MacOS typecode).
- **Outputs/Return:** `true` if successful; `false` if allocation failed or WAD could not be created.
- **Side effects (global state, I/O, alloc):** Allocates `prefInfo` on heap (try-catch). Calls `load_preferences()`. May create/delete/truncate file on disk. Sets game errors on failure.
- **Calls (direct calls visible in this file):** `load_preferences()`, `create_empty_wad()`, `error_pending()`, `get_game_error()`, `set_game_error()`, `w_write_preferences_file()`.
- **Notes:** On corruption (invalid WAD), deletes file and creates fresh empty WAD. Memory allocation errors caught and reported via game error system.

### w_get_data_from_preferences
- **Signature:** `void *w_get_data_from_preferences(WadDataType tag, size_t expected_size, prefs_initializer initialize, prefs_validater validate)`
- **Purpose:** Retrieve preference data, optionally initializing and validating. Caller must not free the returned pointer.
- **Inputs:** `tag` (identifier), `expected_size` (expected byte count), `initialize` (optional callback), `validate` (optional callback).
- **Outputs/Return:** Pointer to preference data in the WAD.
- **Side effects (global state, I/O, alloc):** May allocate and append to `prefInfo->wad`; does not write to disk.
- **Calls (direct calls visible in this file):** `extract_type_from_wad()`, `append_data_to_wad()`, `malloc()`, `free()`, `memcpy()`.
- **Notes:** If data missing or size mismatch, calls `initialize()` and appends. If `validate()` returns true (data was fixed), re-appends fixed data to WAD. Returned pointer is owned by WAD, not caller.

### w_write_preferences_file
- **Signature:** `void w_write_preferences_file(void)`
- **Purpose:** Serialize the in-memory WAD to disk; recreate file to avoid corruption mid-write.
- **Inputs:** (none; uses global `prefInfo`).
- **Outputs/Return:** (void).
- **Side effects (global state, I/O, alloc):** Deletes and recreates preferences file on disk. Clears any pending game error.
- **Calls (direct calls visible in this file):** `error_pending()`, `set_game_error()`, `fill_default_wad_header()`, `open_wad_file_for_writing()`, `write_wad_header()`, `calculate_wad_length()`, `set_indexed_directory_offset_and_length()`, `write_wad()`, `write_directorys()`, `close_wad_file()`.
- **Notes:** Safe to call at exit (checks for pending error first). Assumes WAD directory fits in one entry (index 0).

### load_preferences (static)
- **Signature:** `static void load_preferences(void)`
- **Purpose:** Read WAD from disk into `prefInfo->wad`.
- **Inputs:** (none; uses `prefInfo->PrefsFile`).
- **Outputs/Return:** (void).
- **Side effects (global state, I/O, alloc):** Allocates/replaces `prefInfo->wad`. Calls game error system on failure (e.g., unknown WAD version).
- **Calls (direct calls visible in this file):** `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `close_wad_file()`, `set_game_error()`.
- **Notes:** Frees old WAD before loading new one. Gracefully degrades if WAD version unknown (sets error but does not crash).

## Control Flow Notes
**Startup:** `w_open_preferences_file()` ΓåÆ `load_preferences()` ΓåÆ read from disk or create empty.

**Runtime:** `w_get_data_from_preferences()` called repeatedly to retrieve/initialize individual preference entries; modified entries are updated in-memory WAD.

**Shutdown:** `w_write_preferences_file()` serializes to disk.

Exception handling wraps allocation; game error system used for I/O and version failures instead of exceptions.

## External Dependencies
- **WAD I/O:** `create_empty_wad()`, `open_wad_file_for_reading()`, `open_wad_file_for_writing()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `write_wad_header()`, `write_wad()`, `write_directorys()`, `extract_type_from_wad()`, `append_data_to_wad()`, `calculate_wad_length()`, `close_wad_file()`, `free_wad()` (all defined elsewhere, likely `wad.cpp`).
- **File abstraction:** `FileSpecifier` (OO file spec; methods: `SetToPreferencesDir()`, `Exists()`, `Create()`, `Delete()`, `GetError()`, `GetType()`), `OpenedFile` (opaque file handle).
- **Error system:** `error_pending()`, `get_game_error()`, `set_game_error()` (defined in `game_errors`).
- **Standard library:** `<cstring>`, `<cstdlib>`, `<stdexcept>` for memory and string utilities.
