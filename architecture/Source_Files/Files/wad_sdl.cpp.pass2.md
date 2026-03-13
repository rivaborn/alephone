# Source_Files/Files/wad_sdl.cpp - Enhanced Analysis

## Architectural Role

This file implements **file discovery and verification** for the WAD asset system, enabling the engine to locate game data files (maps, physics, shapes, sounds) by verifiable criteria rather than by name. It bridges the abstract asset resolution layer with concrete filesystem + Steam Workshop enumeration, acting as a critical gatekeeper during map loading and asset synchronization. The file exemplifies the engine's phased modularization: original Marathon code did simple path searching, but wad_sdl.cpp adds checksum-based integrity verification and Steam Workshop integration without restructuring the core WAD parsing (wad.cpp).

## Key Cross-References

### Incoming (who calls these functions)
- **game_wad.cpp** (game state persistence) ΓÇô likely calls `find_wad_file_that_has_checksum()` during map loading to resolve map references by their known checksums
- **Network subsystem** ΓÇô map checksums are exchanged during multiplayer handshakes; file discovery ensures client/server asset parity
- **Preferences/Map loading** ΓÇô initial game startup queries WAD files to populate available maps

### Outgoing (what this file depends on)
- **find_files.h :: FileFinder** ΓÇô base class; provides recursive directory traversal infrastructure via `Find(DirectorySpecifier, Typecode)` 
- **FileHandler.h** ΓÇô `FileSpecifier`, `OpenedFile`, `DirectorySpecifier` abstractions; binary file I/O via SDL_RWops
- **shell_sdl.cpp :: data_search_path** ΓÇô global vector of local directories to search (built at startup from prefs, data folders, plugins)
- **steamshim_child.h :: subscribed_workshop_items** ΓÇô external vector of Steam Workshop items available for search (populated by Steam background thread)
- **SDL2/SDL_endian.h :: SDL_ReadBE32()** ΓÇô endian-safe big-endian integer reading

## Design Patterns & Rationale

**Template Method + Strategy Combo:**  
Both `FindByChecksum` and `FindByDate` inherit from `FileFinder`, overriding the `found()` callback to define their match predicate. This avoids duplicating the directory traversal logic (`Find()` method) while parameterizing the search criteriaΓÇöeach subclass is a one-method struct encoding a single matching rule.

**Two-Tier Search (Steam ΓåÆ Local):**  
The functions check Steam Workshop items first (`#ifdef HAVE_STEAM`), then fall back to `data_search_path`. This prioritizes user-subscribed content while maintaining compatibility for offline/non-Steam distributions. If Steam fails or is disabled, local paths are automatically triedΓÇöa defensive, fail-safe design that anticipates partial platform availability.

**Unused Parameter (`path_resource_id`):**  
The parameter is present but never used. This likely reflects legacy Marathon 1 / MacOS resource fork APIs that indexed files by resource ID. It's retained for ABI stability across the engine, allowing future extensions or backwards-compatibility layers without signature changes. Modern code can safely ignore it.

## Data Flow Through This File

1. **Initialization Phase:**
   - Engine boots; `data_search_path` is populated from `shell_sdl.cpp` (user's data folder, install folder, plugins)
   - If `HAVE_STEAM`, `subscribed_workshop_items` vector is asynchronously populated by Steam background thread

2. **Asset Resolution Phase (e.g., map loading):**
   - Caller invokes `find_wad_file_that_has_checksum(checksum)` or `find_file_with_modification_date(date)` 
   - For each search:
     - **Steam loop** (if available): Iterate subscribed items, filter by type, call `finder.Find(workshop_dir, file_type)` to recursively scan
     - **Local loop**: Iterate `data_search_path` directories in order, call `finder.Find(dir, file_type)` for each
   - `FileFinder::Find()` recursively descends directory trees, calling the finder's `found()` callback on each file
   - When a match is found (checksum matches or date matches), `FileSpecifier` is stored and search short-circuits

3. **Verification Mechanics:**
   - `FindByChecksum::found()` opens the file, seeks to byte 0x44, reads big-endian uint32, compares
   - `FindByDate::found()` queries file metadata via `FileSpecifier::GetDate()`, compares timestamps
   - If match, return `true`; `FileFinder` stops iterating and populates `found_what`

## Learning Notes

**Marathon WAD Format Convention:** The hardcoded offset 0x44 for checksum is a Marathon serialization detailΓÇöbinary game formats of this era rarely evolved gracefully, so offsets are baked into search logic rather than extracted from headers. Modern engines would include offset metadata in a file header or use cryptographic hashing instead.

**Iterative C++ (not STL idioms):** The local path loop uses `vector::const_iterator` with manual increment (`i++`), bypassing range-based `for` loops. This reflects mid-2000s C++ code style before C++11 idiom adoption. The steam loop does use range-based `for`, suggesting incremental modernization of this file.

**Conditional Compilation Overhead:** The `#ifdef HAVE_STEAM` guards scatter platform-specific code throughout the functions. A modern refactor might extract Steam searching into a separate function or virtual interface, reducing preprocessor usage and improving testability.

**Parameter Shadowing:** Both `typecode_to_item_type()` calls inside the `#ifdef` block convert file type ΓåÆ item type, but this function is also `#ifdef HAVE_STEAM`-gated. If Steam is disabled, callers still pass `file_type` but it goes unusedΓÇöa code smell suggesting this function pair was added as a monolithic feature block.

## Potential Issues

1. **Code Duplication:** `find_wad_file_that_has_checksum()` and `find_file_with_modification_date()` are 90% identical (only the finder class and `typecode_to_item_type()` differ). A refactor could extract `find_in_paths_generic(FileFinder&, Typecode)` to reduce maintenance burden.

2. **Assert on Unknown Typecode:** `typecode_to_item_type()` asserts on unmapped typecodes, crashing release builds if an unexpected file type is passed. Should either add a default case or throw a recoverable exception.

3. **No Timeout/Recursion Limit:** `FileFinder::Find()` (from find_files.h) recursively descends directories. If a malicious or misconfigured workshop folder contains symlink loops, the search could hang or stack-overflow. The cross-reference index doesn't provide find_files.h details, so this can't be fully assessed here.

4. **Steam Item Path Trust:** `item.install_folder_path` from Steam is trusted without validation. If Steam's directory pointer is corrupted or points outside the intended scope, `finder.Find()` could enumerate unintended system directories.
