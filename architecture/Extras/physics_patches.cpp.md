# Extras/physics_patches.cpp

## File Purpose
A command-line utility that compares two physics WAD files and generates a delta/patch file containing only the byte-level differences. The patch can later be applied to the original file to reconstruct the new version, preserving the original file's checksum in the patch metadata.

## Core Responsibilities
- Parse command-line arguments (original file path, new file path, output patch file path)
- Open and read physics WAD files from disk
- Extract physics definition data from WAD structures
- Perform byte-by-byte comparison across all physics definition types
- Identify changed regions (first non-matching offset, contiguous length)
- Generate a delta WAD file containing only changed data with offset metadata
- Write patch file with parent checksum linkage for validation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileDesc` | typedef | File descriptor (defined elsewhere; used for cross-platform file I/O) |
| `FSSpec` | struct | Mac OS file specification (volume, directory, filename) |
| `wad_data` | struct | In-memory WAD structure containing tag data arrays |
| `wad_header` | struct | WAD file header with version, checksum, directory offset metadata |
| `directory_entry` | struct | WAD directory entry mapping tag index to file offset and length |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `monster_definitions` | byte[1] | global | Placeholder array to satisfy header inclusion (actual data in WAD) |
| `projectile_definitions` | byte[1] | global | Placeholder array (actual data in WAD) |
| `effect_definitions` | byte[1] | global | Placeholder array (actual data in WAD) |
| `weapon_definitions` | byte[1] | global | Placeholder array (actual data in WAD) |
| `physics_models` | byte[1] | global | Placeholder array (actual data in WAD) |
| `create_physics_file` | static fn | file | Forward declaration; not implemented |
| `create_delta_wad_file` | static fn | file | Forward declaration; core worker function |

## Key Functions / Methods

### main
- **Signature:** `void main(int argc, char **argv)`
- **Purpose:** Entry point; validates arguments and orchestrates patch creation.
- **Inputs:** `argc` (argument count), `argv` (file path strings: original, new, patch output)
- **Outputs/Return:** Exit code (0 on success, 1 on error); writes patch file to disk.
- **Side effects:** Calls `FSMakeFSSpec()` (Mac-specific), `create_delta_wad_file()`.
- **Calls:** `FSMakeFSSpec()`, `c2pstr()`, `create_delta_wad_file()`, `fprintf()`, `exit()`.
- **Notes:** Expects exactly 3 command-line arguments; tolerates missing patch file (`fnfErr`). Uses `temporary` global (from headers).

### get_physics_wad_from_file
- **Signature:** `static struct wad_data *get_physics_wad_from_file(FileDesc *file, unsigned long *checksum)`
- **Purpose:** Open a physics WAD file, read its header, extract physics data (index 0), and retrieve the file checksum.
- **Inputs:** `file` (file descriptor), `checksum` (pointer to store checksum or NULL).
- **Outputs/Return:** Pointer to `wad_data` structure (or NULL on error); checksum written to output parameter if non-NULL.
- **Side effects:** Opens/closes file descriptor; prints error messages to `stderr`.
- **Calls:** `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `close_wad_file()`, `fprintf()`.
- **Notes:** Validates `data_version == PHYSICS_DATA_VERSION`; returns NULL if any step fails.

### write_patchfile
- **Signature:** `static boolean write_patchfile(FileDesc *file, struct wad_data *wad, short data_version, unsigned long parent_checksum, unsigned long patch_filetype)`
- **Purpose:** Create a new WAD file containing the delta data and link it to the original via parent checksum.
- **Inputs:** `file` (output file), `wad` (delta data to write), `data_version` (PHYSICS_DATA_VERSION), `parent_checksum` (original file's checksum), `patch_filetype` (file type constant).
- **Outputs/Return:** `TRUE` if write succeeds, `FALSE` on error. Writes WAD file to disk.
- **Side effects:** Creates/overwrites file; calculates and stores WAD checksum.
- **Calls:** `create_wadfile()`, `open_wad_file_for_writing()`, `fill_default_wad_header()`, `write_wad_header()`, `write_wad()`, `calculate_wad_length()`, `write_directorys()`, `calculate_and_store_wadfile_checksum()`, `close_wad_file()`.
- **Notes:** Updates header twice (once after data, once after directory). Parent checksum field enables patch validation.

### create_delta_wad_file
- **Signature:** `static boolean create_delta_wad_file(FileDesc *original, FileDesc *new, FileDesc *delta, unsigned long delta_file_type)`
- **Purpose:** Core comparison function; reads both WAD files, compares all physics data types, and generates a delta WAD.
- **Inputs:** `original` (original physics file), `new` (new physics file), `delta` (output patch file), `delta_file_type` (patch file type constant).
- **Outputs/Return:** `TRUE` if patch created or files are identical, `FALSE` on error. Writes patch file to disk.
- **Side effects:** Calls `fprintf()` for diagnostic output; allocates/frees WAD structures.
- **Calls:** `get_physics_wad_from_file()`, `create_empty_wad()`, `extract_type_from_wad()`, `append_data_to_wad()`, `write_patchfile()`, `free_wad()`.
- **Notes:** 
  - Loops through `NUMBER_OF_DEFINITIONS` (count of all physics definition types).
  - For each definition, extracts data by tag and compares byte-by-byte.
  - Asserts both files have identical structure (same lengths for each tag).
  - Tracks first non-matching offset and contiguous non-matching region.
  - Outputs diagnostic lines for each definition (tag, offset, length).
  - Returns TRUE even if files are identical (no error).

### level_transition_malloc
- **Signature:** `void *level_transition_malloc(size_t size)`
- **Purpose:** Wrapper around `malloc()` (appears unused; likely legacy code).
- **Inputs:** `size` (bytes to allocate).
- **Outputs/Return:** Pointer to allocated memory or NULL.
- **Calls:** `malloc()`.
- **Notes:** Dead code; never called in this file.

## Control Flow Notes
**Initialization:** None (utility, not engine code).

**Frame/Update:** N/A.

**Main Logic:**
1. `main()` parses arguments and creates `FSSpec` structures.
2. `create_delta_wad_file()` orchestrates comparison:
   - Opens original and new physics files.
   - Iterates through each physics definition type.
   - Compares byte ranges; tracks differences.
   - Appends delta regions to a new WAD.
   - Writes patch WAD with parent checksum.
3. Exit with status code.

**Shutdown:** Files closed via `close_wad_file()` in `get_physics_wad_from_file()`.

## External Dependencies
- **Headers:** `macintosh_cseries.h` (Mac OS compat), `map.h`, `effects.h`, `projectiles.h`, `monsters.h`, `weapons.h`, `wad.h`, `items.h`, `mysound.h`, `media.h`, `tags.h`
- **WAD Functions (defined elsewhere):**
  - `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `close_wad_file()`
  - `extract_type_from_wad()`, `create_empty_wad()`, `append_data_to_wad()`, `free_wad()`
  - `create_wadfile()`, `open_wad_file_for_writing()`, `fill_default_wad_header()`, `write_wad_header()`, `write_wad()`, `write_directorys()`, `calculate_and_store_wadfile_checksum()`, `calculate_wad_length()`
- **Mac OS Functions:** `FSMakeFSSpec()`, `c2pstr()` (C string to Pascal string).
- **Standard C:** `fprintf()`, `strcpy()`, `exit()`, `malloc()`.
- **Macros:** `definitions[]` array (populated via included headers), `NUMBER_OF_DEFINITIONS` (count constant), `PATCH_FILE_TYPE`, `PHYSICS_DATA_VERSION`, `CURRENT_WADFILE_VERSION`.
