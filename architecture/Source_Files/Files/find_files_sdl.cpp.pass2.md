# Source_Files/Files/find_files_sdl.cpp - Enhanced Analysis

## Architectural Role

This file implements the **file discovery subsystem** for Aleph One's resource loading pipeline. It bridges the platform-agnostic `FileHandler` abstraction with concrete directory traversal logic, enabling the engine to discover game resources (scenarios, plugins, media assets) without hard-coding file paths. The breadth-first design and typecode filtering integrate directly with the Files subsystem's broader role in resource abstraction, supporting both single-file lookups and batch discovery patterns needed during initialization and plugin enumeration.

## Key Cross-References

### Incoming (who depends on this file)

- **`find_files.h`**: Defines `FileFinder` base class and `FindAllFiles` derived class interface
- **Plugin system** (`Source_Files/Misc` or `Source_Files/XML`): Likely calls `FindAllFiles::Find()` to enumerate available plugins at startup
- **Scenario/map loader** (`Source_Files/Files/game_wad.cpp`): Probably uses file discovery to locate `.sceA` / WAD files in standard directories
- **Steam Workshop integration**: May invoke discovery to scan Workshop directories for community content
- **Preferences system** (`Source_Files/Files/wad_prefs.cpp`): Could use typecode-filtered discovery for preferences files

### Outgoing (what this file depends on)

- **`FileHandler.h`**: `FileSpecifier`, `DirectorySpecifier` (typedef alias), `dir_entry` struct, `ReadDirectory()` method
- **`find_files.h`**: Base class `FileFinder`, derived class `FindAllFiles`, `WILDCARD_TYPE` constant
- **`cseries.h`**: Platform abstraction, header chain for portability macros
- **STL**: `<queue>`, `<vector>`, `<algorithm>` for container operations and sorting

## Design Patterns & Rationale

**Breadth-First Search (BFS) via Queue**
- Uses `std::queue<std::pair<int, FileSpecifier>>` instead of recursion
- **Why**: Prevents stack overflow on pathologically deep directory trees; maintains per-level ordering for deterministic output
- **Tradeoff**: Slightly more code than recursive DFS, but safer for untrusted filesystem layouts

**Virtual Callback (Strategy/Iterator Pattern)**
- `found(FileSpecifier&)` is virtual; subclasses override to define result handling
- **Why**: Pre-C++11 idiom (code from ~2000); avoids template complexity for callers
- **Tradeoff**: Cannot use lambdas/`std::function`; requires subclass instantiation; early exit via return value instead of exception

**Explicit Depth Tracking**
- Maintains `depth` counter; only filters at `depth == 0` (root level)
- **Why**: Allows future extension (max-depth limits, different filtering per level); prevents unintended filtering of "Plugins" subdirectories if nested
- **Rationale**: Shows defensive designΓÇödevelopers anticipated future need for depth-aware logic

**Sorted Directory Entries**
- Calls `std::sort(entries.begin(), entries.end())` each iteration
- **Why**: Deterministic, reproducible ordering (relies on `dir_entry::operator<`); important for reproducible behavior in multiplayer/demo playback
- **Idiomatic**: Aleph One prioritizes determinism (echoes of Marathon's network-play era)

## Data Flow Through This File

```
Input:
  DirectorySpecifier dir ΓåÆ starting root
  Typecode type         ΓåÆ filter (WILDCARD_TYPE = match all)
  bool recursive        ΓåÆ descend subdirectories?

Pipeline:
  1. Initialize: queue.push((depth=0, dir))
  
  2. BFS Loop:
     For each (depth, directory) in queue:
       a. ReadDirectory(entries) ΓåÆ sorted list of files + subdirs
       b. std::sort(entries)     ΓåÆ deterministic ordering
       c. For each entry:
          - If directory:
              Filter: skip "Plugins"/"Scenarios" if depth == 0
              If recursive: queue.push((depth+1, entry))
          - If file:
              Filter: type == WILDCARD_TYPE OR type == file.GetType()
              Action: found(file) ΓåÆ callback returns (continue? or abort?)
  
  3. Output: bool (true=aborted early, false=completed/error)
           Side effects: caller's callback modifies state (e.g., appends to vector)
```

**State Flow in FindAllFiles::found()**:
- Called once per matching file
- Appends file to `dest_vector` (caller's container)
- Always returns `false` ΓåÆ search continues to completion

## Learning Notes

1. **Pre-Lambda Callback Era**: Virtual methods as the primary callback mechanism; modern code would use `std::function<>` or lambdas. This reveals Aleph One's long codebase age and commitment to backward compatibility.

2. **Determinism-First Design**: Explicit sorting each iteration signals that reproducible output mattersΓÇölikely because:
   - Network multiplayer requires identical file discovery on all peers
   - Demo/replay playback must find resources in the same order
   - This is an artifact of Marathon's legacy (1994 networked gameplay)

3. **Conservative Directory Traversal**: Hard-coded exclusion of "Plugins" and "Scenarios" at root-only suggests a "whitelist-safe" designΓÇödiscovering scenarios in `Scenarios/Maps/` (nested) but not `Scenarios/` itself, preventing confusion.

4. **Integration with Legacy Resource Model**: The use of typecodes (Marathon's OS-era concept similar to Mac Classic resource types) shows this engine maintains compatibility with 1990s game design patterns. Modern engines use MIME types / file extensions.

5. **BFS Over Recursion**: Unusual choice for 2000-era code when recursion was standard. Suggests the developers anticipated either very deep directory structures or sought to simplify stack frame management (possible embedded/console origin).

## Potential Issues

1. **Depth Tracking Underutilized**: `depth` is incremented but only checked once. If max-depth was intended, it's missingΓÇödirectories could be traversed arbitrarily deep, causing slow startup if user has a pathological filesystem structure.

2. **Hard-Coded Path Exclusions**: "Plugins" and "Scenarios" filtering is brittle. If the engine layout changes or users rename directories, discovery fails silently. Could be parameterized.

3. **No Exception Safety**: If `found()` callback throws, the queue leaks (though RAII cleanup via automatic destruction mitigates this in modern C++). Pre-C++11 code may not assume exception safety.

4. **Silent Read Failures**: `ReadDirectory()` returning `false` causes `Find()` to return `false` immediately, indistinguishable from "no files found." Callers cannot distinguish IO errors from legitimate empty results.

5. **Sorting Overhead**: Re-sorting every directory level is O(n*k log k) where n=directory depth, k=entries per level. For large flat structures, this could be slow. One-time sort might be cheaper.

---

**Sources of Insight**: 
- Architecture overview showing Files subsystem responsibilities
- Code era and style (C++98/03 + recent STL) suggesting early-2000s origin
- Integration points with plugin, scenario, and workshop systems implied by directory filtering
- Cross-reference index showing callers and dependencies
