# Source_Files/Files/find_files_sdl.cpp

## File Purpose
Implements recursive file discovery for the Aleph One game engine's SDL backend. Provides breadth-first directory traversal with typecode-based file filtering and callback-driven result collection.

## Core Responsibilities
- Implements breadth-first recursive directory traversal using a queue
- Filters files by typecode (or returns all files if `WILDCARD_TYPE` specified)
- Skips engine-specific directories ("Plugins", "Scenarios") at root level
- Invokes virtual callback method for each matching file found
- Implements concrete callback for `FindAllFiles` to collect results into a vector

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `std::queue<std::pair<int, FileSpecifier>>` | template instantiation | BFS queue holding (depth, directory) pairs for traversal |
| `std::vector<dir_entry>` | template instantiation | Temporary container for sorted directory entries per level |
| `dir_entry` | struct (from FileHandler.h) | Directory/file entry metadata (name, is_directory, date) |

## Global / File-Static State
None.

## Key Functions / Methods

### FileFinder::Find
- **Signature:** `bool Find(DirectorySpecifier& dir, Typecode type, bool recursive)`
- **Purpose:** Search recursively for all files matching a typecode within a directory tree.
- **Inputs:**
  - `dir`: Starting directory to search
  - `type`: Target typecode (or `WILDCARD_TYPE` for all files)
  - `recursive`: Whether to descend into subdirectories
- **Outputs/Return:** `true` if search was aborted by callback, `false` if completed or error occurred
- **Side effects:** 
  - Reads filesystem (via `ReadDirectory()`)
  - Invokes `found()` callback for each matching file
  - May modify callback's state (e.g., appends to `dest_vector` in `FindAllFiles`)
- **Calls:** 
  - `directory.ReadDirectory(entries)` ΓÇô retrieves sorted directory contents
  - `std::sort(entries.begin(), entries.end())` ΓÇô sorts entries
  - `found(file)` ΓÇô virtual callback (pure in base class)
- **Notes:**
  - Uses BFS (queue) to maintain breadth-first ordering
  - Explicitly skips "Plugins" and "Scenarios" only at depth 0
  - Directory entries are pre-sorted by `dir_entry::operator<` (directories before files)
  - Returns early (`true`) if callback signals abort

### FindAllFiles::found
- **Signature:** `bool found(FileSpecifier &file)` (override)
- **Purpose:** Collect matching file into result vector.
- **Inputs:** `file` ΓÇô matching FileSpecifier to collect
- **Outputs/Return:** Always `false` (continue search)
- **Side effects:** Appends `file` to `dest_vector`
- **Calls:** (none)
- **Notes:** Simple accumulator callback; never aborts search.

## Control Flow Notes
Operates during initialization/file-discovery phase. Uses BFS to traverse the file tree level-by-level, allowing callers to inspect results progressively if implementing custom callbacks. The virtual `found()` method acts as an iterator callback, enabling both collection (`FindAllFiles`) and early-exit patterns.

## External Dependencies
- **FileHandler.h:** `FileSpecifier`, `DirectorySpecifier` (typedef alias), `dir_entry`
- **find_files.h:** `FileFinder` base class, `FindAllFiles` derived class, `WILDCARD_TYPE` constant
- **cseries.h:** Portability macros and includes
- **STL:** `<vector>`, `<algorithm>`, `<queue>`
