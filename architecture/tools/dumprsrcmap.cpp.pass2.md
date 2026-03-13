# tools/dumprsrcmap.cpp - Enhanced Analysis

## Architectural Role

This is a **standalone offline inspection utility** for Macintosh resource maps, operating outside the main engine runtime loop. It reuses the engine's resource fork abstraction layer (`resource_manager.cpp`) to transparently handle legacy Marathon format detection (raw fork, AppleSingle, MacBinary), allowing developers and data packagers to audit resource metadata without loading the full game. The utility demonstrates the engine's intentional separation of resource *parsing* logic (format-agnostic in resource_manager) from resource *usage* logic (in the main engine), enabling such tools to exist as thin CLI wrappers around the core subsystem.

## Key Cross-References

### Incoming
- **None (standalone tool)**. This is a one-shot CLI utility; no engine code calls it. It exists for offline inspection during development/packaging, not runtime integration.

### Outgoing
- **`resource_manager.cpp`** (included as `.cpp`, not linked) ΓÇö calls `open_res_file()`, `cur_res_file_t`, `res_file_t::types`, `count_resources()`, `get_resource_id_list()`, `get_resource()`, `LoadedResource::GetLength()`
- **`FileHandler_SDL.cpp`** (included as `.cpp`) ΓÇö pulls in `FileSpecifier` and `open_res_file()` implementation
- **`csalerts_sdl.cpp`, `Logging.cpp`, `XML_ElementParser.cpp`** (included as `.cpp`) ΓÇö satisfy linker dependencies of `resource_manager.cpp`; unused directly in this file but required for compilation
- **Global stubs** ΓÇö provides dummy `set_game_error()` and file-path globals (`local_data_dir`, etc.) to avoid link errors from `resource_manager.cpp` references

---

## Design Patterns & Rationale

### Monolithic Compilation (Early 2000s Pattern)
The tool includes `.cpp` files directly (`#include "resource_manager.cpp"`) rather than linking precompiled objects. This bundling strategy:
- **Avoids build complexity**: No separate library compilation step; single `g++ tools/dumprsrcmap.cpp -o dumprsrcmap` works.
- **Couples to internals**: Changes to `resource_manager.cpp` force recompilation; tight coupling to implementation, not interface.
- **Reflects era**: Common in Marathon-era codebases (early 2000s) before modern CMake/build systems.

### Stub Linking Strategy
Rather than pulling in the entire engine, the utility provides minimal stubs:
- `set_game_error(short, short)` ΓÇö no-op silent failure handler (engine sets errors via this; tool ignores them)
- Four global `DirectorySpecifier` stubs ΓÇö satisfy external declarations without initializing the filesystem hierarchy
- `vector<DirectorySpecifier> data_search_path` ΓÇö stub for resource search paths (unused in this context)

This prevents cascading dependencies on preferences, logging infrastructure, UI subsystems, etc.

### Resource Type as Fourcc
Resource types are 4-byte codes (e.g., `'WMAP'`, `'PARM'`). Printing as `'%c%c%c%c'` + hex provides both human-readable form and binary value for scripting. This idiom is native to Classic Mac OS resource managers.

---

## Data Flow Through This File

```
argv[1] (file path string)
    Γåô
FileSpecifier (cross-platform path wrapper)
    Γåô
open_res_file() ΓåÆ SDL_RWops + parses fork header
    Γåô
*cur_res_file_t (global: res_file_t struct in memory)
    Γö£ΓöÇ types: map<uint32, id_map_t> (resource type ΓåÆ IDs)
    Γöé
    Γö£ΓöÇ for each type: iterate res_file_t::types.begin()..end()
    Γöé   Γö£ΓöÇ uint32 type = titΓåÆfirst
    Γöé   Γö£ΓöÇ count_resources(type) ΓåÆ int
    Γöé   Γö£ΓöÇ printf("'%c%c%c%c' (...) %d resources:\n")
    Γöé   Γöé
    Γöé   ΓööΓöÇ get_resource_id_list(type, ids) ΓåÆ vector<int>
    Γöé       ΓööΓöÇ for each ID in ids:
    Γöé           Γö£ΓöÇ get_resource(type, ID) ΓåÆ LoadedResource
    Γöé           Γö£ΓöÇ rsrc.GetLength() ΓåÆ size in bytes
    Γöé           Γö£ΓöÇ printf("  id %d, size %d\n")
    Γöé           ΓööΓöÇ rsrc.Unload() (immediately free memory)
    Γöé
    ΓööΓöÇ exit(0)
    
Outputs to stdout; no persistent state retained
```

**Key insight**: **Stream-through pattern**ΓÇöeach resource is loaded, inspected, and unloaded immediately. No accumulation; minimal memory overhead.

---

## Design Patterns & Rationale

### Why Immediate Unload?
The `Unload()` call on each resource (line 62) frees memory before moving to the next. This prevents quadratic memory growth if a WAD has thousands of resources. The tool is designed to handle large archives by sacrificing random access for memory efficiency.

### Why No Explicit File Close?
The tool never explicitly closes `rw` (SDL_RWops). This works because:
- Modern OSes clean up file descriptors on process exit
- The tool is single-use, not a daemon
- However, this violates RAII principles and would be flagged in modern code reviews

---

## Learning Notes

Developers studying this file learn:

1. **Resource fork abstraction is transparent**: Code doesn't care if the file is raw Mac fork, AppleSingle, or MacBinaryΓÇöthe resource manager handles format detection. This is crucial for Marathon compatibility on modern platforms.

2. **Global state is acceptable in 2000s C++**: The `cur_res_file_t` global pointer is set by `open_res_file()` and assumed valid thereafter. Modern engines use RAII or dependency injection; this era used globals freely.

3. **Fourcc codes are idiomatic to game formats**: Marathon stored entity data, weapon definitions, physics, etc., under type codes like `'MONP'` (monster physic), `'WEAP'` (weapon), `'PXS'` (projectile effects). The tool makes these visible.

4. **Monolithic compilation is pragmatic but costly**: Bundling `.cpp` files in a single unit simplifies build scripting but couples tools to implementation internals. Modern engines separate tools via libraries.

5. **Resource inspection is a first-class operation**: Aleph One treats dumping/inspecting archives as important as playing. This reflects Marathon's design: mods are packages of resources, not code, so tooling to examine them is essential.

---

## Potential Issues

1. **Dangling pointer risk** (line 47): `*cur_res_file_t` is dereferenced without post-`open_res_file()` validation. If `open_res_file()` sets the pointer to garbage (e.g., on partial parse failure), line 48 will crash. No null-check.

2. **Silent error loss** (line 51ΓÇô57): `get_resource()` failures are not checked; `GetLength()` may return 0 or garbage if loading fails. The stub `set_game_error()` swallows errors, so malformed resources print as size 0 with no diagnostic.

3. **Resource leak on early exit** (line 42ΓÇô45): If file open fails, the program exits without closing `rw`. On most OSes this is benign (process cleanup), but it's sloppy resource discipline.

4. **Unused includes** (line 9): `XML_ElementParser.cpp` is pulled in (likely a copy-paste from resource_manager includes) but never called. Creates unnecessary dependencies.

5. **No endianness awareness in output**: Resource type printing assumes the CPU is big-endian or data is already byte-swapped. On little-endian systems, `'%c%c%c%c' type >> 24, type >> 16, type >> 8, type` may print backwards.
