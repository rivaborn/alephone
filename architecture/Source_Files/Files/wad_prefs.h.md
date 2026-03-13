# Source_Files/Files/wad_prefs.h

## File Purpose
Header for a preferences management system that reads and writes game configuration data using the WAD file format. Provides initialization, validation, and serialization of typed preference data through callback-based registration.

## Core Responsibilities
- Open and manage preferences files (FileSpecifier-based)
- Load typed preference data with custom initialization and validation callbacks
- Persist preferences back to the WAD file
- Maintain internal structure linking file specifier to WAD data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `preferences_info` | struct | Internal state container holding the preferences file path and loaded WAD data |
| `prefs_initializer` | typedef (function pointer) | Callback to initialize preference data if allocation is needed |
| `prefs_validater` | typedef (function pointer) | Callback to validate and repair preference data; returns whether valid |

## Global / File-Static State
None.

## Key Functions / Methods

### w_open_preferences_file
- **Signature:** `bool w_open_preferences_file(char *PrefName, Typecode Type)`
- **Purpose:** Open or create a preferences file and prepare internal structures for data access
- **Inputs:** Preference filename (string), file typecode
- **Outputs/Return:** Boolean success status
- **Side effects:** Allocates and initializes the internal `preferences_info` structure; I/O to filesystem
- **Calls:** (not visible in header; defined elsewhere)
- **Notes:** Called at startup; must be called before `w_get_data_from_preferences()`

### w_get_data_from_preferences
- **Signature:** `void *w_get_data_from_preferences(WadDataType tag, size_t expected_size, prefs_initializer initialize, prefs_validater validate)`
- **Purpose:** Retrieve typed preference data from the loaded WAD, with deferred initialization and validation
- **Inputs:** Data tag (WadDataType), expected size, initializer callback, validator callback
- **Outputs/Return:** Void pointer to preference data (or NULL if not found/invalid)
- **Side effects:** Allocates memory if initializer is called; may repair data via validate callback
- **Calls:** (not visible; uses callbacks)
- **Notes:** Generic data retrieval; caller is responsible for type casting. Validator can modify data in-place.

### w_write_preferences_file
- **Signature:** `void w_write_preferences_file(void)`
- **Purpose:** Flush all loaded preferences back to the WAD file
- **Inputs:** None
- **Outputs/Return:** Void
- **Side effects:** I/O to filesystem; modifies or creates preferences file
- **Calls:** (not visible; defined elsewhere)
- **Notes:** No return value; errors are internal

## Control Flow Notes
This module is part of the game's initialization and configuration subsystem. Typical flow: `w_open_preferences_file()` ΓåÆ repeated calls to `w_get_data_from_preferences()` during init ΓåÆ `w_write_preferences_file()` on shutdown or config change. The use of callbacks (`prefs_initializer`, `prefs_validater`) allows decoupled registration of preference schemas.

## External Dependencies
- **FileHandler.h:** `FileSpecifier` class for file/directory abstraction
- **wad.h:** `wad_data` struct, `WadDataType` typedef, WAD file I/O functions (`read_indexed_wad_from_file`, `free_wad`, etc.)
- **tags.h** (via wad.h): Type code constants (`Typecode`)
