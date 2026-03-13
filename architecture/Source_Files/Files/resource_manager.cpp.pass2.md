# Source_Files/Files/resource_manager.cpp - Enhanced Analysis

## Architectural Role

This file is the **legacy format compatibility layer** enabling Aleph One to read Marathon's original Mac resource forks on Linux/Windows/modern macOS. It sits at the boundary between the file I/O subsystem (FileHandler) and higher-level resource consumers (rendering, audio, etc.), transparently adapting three container formats (AppleSingle, MacBinary II/III, raw fork) into a unified lookup interface. The stack-based multi-file design supports both base game resources and user mod overlays, critical for the modding ecosystem where later-opened files shadow earlier ones.

## Key Cross-References

### Incoming (who depends on this)
- **Shell/Game initialization** ΓÇö calls `open_res_file(FileSpecifier)` to load Marathon 1 data or external resource files
- **Rendering pipeline** (shapes.cpp, textures.cpp, collection_definition.h) ΓÇö uses `get_resource()` / `get_ind_resource()` to load bitmap shapes, palettes, transfer modes
- **Audio subsystem** (Sound/) ΓÇö loads sound definitions and music cues via `count_resources()` / `get_resource()`
- **Terminal/UI system** ΓÇö loads text resources and terminal definitions from external Marathon 1 `.rsrc` files
- **Lua scripting** / XML config ΓÇö may trigger resource lookups during MML parsing for custom assets
- **Preferences/saved state** ΓÇö loads game resources (e.g., player images, color palettes)

### Outgoing (what this file depends on)
- **FileHandler.h** ΓÇö FileSpecifier for path handling, LoadedResource for memory-managed resource data wrapper
- **SDL2** (SDL_endian.h, SDL_RWops) ΓÇö cross-platform file I/O with byte-order utilities
- **Logging.h** ΓÇö logTrace, logNote, logAnomaly, logDump for diagnostic instrumentation
- **CSeries** (cseries.h) ΓÇö platform-agnostic fixed-width types (uint32, int32) and potentially byte_swapping routines
- **No external resource dependencies** ΓÇö intentionally isolated; doesn't depend on GameWorld, Rendering, Audio subsystems, keeping it reusable

## Design Patterns & Rationale

### 1. **Format Abstraction via Detection**
- `is_applesingle()` and `is_macbinary()` detect container formats and extract fork offsets transparently
- **Rationale**: Marathon assets shipped as .rsrc files (Mac resource fork binaries). When ported to non-Mac, these files needed preservation. AppleSingle and MacBinary were standard format-preservation standards. By auto-detecting, the engine accepts .rsrc files in any format without user intervention.

### 2. **Stack-Based File Context (LIFO shadowing)**
- Multiple `res_file_t` objects in `res_file_list`; `cur_res_file_t` points to current (top-of-stack)
- Default `get_resource()` / `count_resources()` **search backwards from current to list.begin()** 
- **Rationale**: Mimics classic MacOS resource fork precedence rules where newly opened resource files shadow older ones. Essential for modding: a mod can open `/path/to/mod.rsrc` after base game, and resource lookups hit the mod version first. When the mod is closed, base game resources become visible againΓÇöno duplicate code needed.

### 3. **Lazy Data Loading with Eager Mapping**
- Resource **map** (typeΓåÆIDΓåÆoffset) is parsed on `open_res_file()` (one-time I/O)
- Resource **data** is only loaded on `get_resource()` call (lazy, on-demand)
- **Rationale**: Maps are small (~KB); fully parsing them once enables fast O(log n) lookups. Actual resource data (images, sounds, text) can be large; loading on-demand reduces peak memory and startup time.

### 4. **Wrapper Overload Pattern**
Three-tier function hierarchy:
```cpp
// Core: single-file lookup
bool res_file_t::get_resource(uint32 type, int id, LoadedResource &rsrc) const

// Current-file: convenience wrapper
bool get_1_resource(uint32 type, int id, LoadedResource &rsrc) { 
    return (*cur_res_file_t)->get_resource(type, id, rsrc); 
}

// Multi-file: search backwards from current
bool get_resource(uint32 type, int id, LoadedResource &rsrc) { 
    // loop from cur_res_file_t backwards
}
```
**Rationale**: Avoids code duplication. Core logic lives in the struct; wrappers add state management (current file) and multi-file semantics.

## Data Flow Through This File

```
1. OPEN PHASE:
   FileSpecifier (base path)
     ΓåÆ open_res_file() tries: .rsrc, .resources, raw file, /..namedfork/rsrc
     ΓåÆ SDL_RWFromFile() + open_res_file_from_rwops()
     ΓåÆ is_applesingle() / is_macbinary() detection
       [if fork_start offset + rsrc_length determined]
     ΓåÆ res_file_t::read_map() parses header + type list + reference lists
       [populates types: map<uint32, map<int, uint32>> with type ΓåÆ {ID ΓåÆ file_offset}]
     ΓåÆ SDL_RWops added to res_file_list, marked as current
     ΓåÆ Returns SDL_RWops handle to caller

2. LOOKUP PHASE:
   (type=PICT, id=128, for example) 
     ΓåÆ get_resource(type, id, LoadedResource&)
     ΓåÆ search cur_res_file backwards to list.begin()
       ΓåÆ (*file)->get_resource(type, id, rsrc)
         ΓåÆ types[type] ΓåÆ id_map[id] ΓåÆ offset_in_file
         ΓåÆ SDL_RWseek(f, offset)
         ΓåÆ read 4-byte big-endian size
         ΓåÆ malloc + SDL_RWread(data)
         ΓåÆ rsrc.p = data, rsrc.size = size
     ΓåÆ return true on success

3. CLEANUP PHASE:
   close_res_file(SDL_RWops *f)
     ΓåÆ find_res_file_t(f) in list
     ΓåÆ erase from list
     ΓåÆ SDL_RWclose(f)
     ΓåÆ delete res_file_t object
     ΓåÆ update cur_res_file_t to last remaining file (or reset)
```

**Resource data layout in file** (after fork_start):
```
[4] data_offset (offset to resource data section)
[4] map_offset  (offset to resource map section)
[4] data_size
[4] map_size
... resource data ...
... resource map: type_list { type, num_refs, ref_list_offset, ... } ...
       ref_list: { id, reserved, data_offset (24-bit), ... } ...
```

## Learning Notes

### What This File Teaches

1. **Legacy Format Archaeology**: Understanding AppleSingle (RFC 1740) and MacBinary structure shows how Mac resources were preserved. The CRC-16 check in `is_macbinary()` is a real-world example of format validation.

2. **Cross-Platform Abstraction Without Overhead**: Uses SDL2's byte-order utilities (`SDL_ReadBE32`, `SDL_WriteBE16`) instead of custom code. Modern engines would use `std::endian` (C++20), but this predates that standard.

3. **Stacking/Shadowing for Modular Assets**: The LIFO search pattern is elegant and powerfulΓÇömods don't need a merge tool; they just open their file and shadow base game resources. Closing the mod file makes base game resources available again. Modern game engines often use asset priority maps; this uses file order.

4. **Graceful Degradation**: Multiple fallback attempts in `open_res_file()` ΓÇö if `.rsrc` fails, try `.resources`, raw file, then Darwin fork path. Shows defensive design for unknown user file layouts.

5. **Deterministic Logging for Debugging**: Calls to `logTrace()`, `logNote()`, `logDump()` at every significant decision point allow diagnosing resource loading failures across platforms without debugger access.

### Idiomatic to This Era

- **Manual memory management**: `new res_file_t()`, `delete r` ΓÇö no smart pointers (pre-C++11 era code)
- **Iterator-based container traversal**: Classic STL `list<>::iterator` and `map<>::find()` patterns
- **Shallow copy semantics**: `res_file_t` copy constructor doesn't deep-clone SDL_RWops; just aliases
- **Static globals for singleton state**: `res_file_list`, `cur_res_file_t` are module-scoped; no class wrapper
- **Endianness conversion at I/O boundaries**: All file reads use `SDL_ReadBE*` explicitly; data is always big-endian in Marathon files

Modern engines would likely:
- Use `std::shared_ptr<Resource>` or handle resources via a ResourceManager class
- Provide thread-safe concurrent resource access (this has none)
- Support async I/O and resource streaming (this is fully synchronous)
- Use a virtual filesystem abstraction for all resource types, not separate code paths

## Potential Issues

### 1. **Resource Data Size Validation Gap**
In `res_file_t::get_resource()` (visible in first-pass but worth highlighting):
```cpp
// Seek to offset, read size, malloc, read data
// But: what if the stored size extends past EOF?
```
A corrupt resource reference list could point to an offset where the 4-byte size overflows, causing `malloc()` of gigabytes followed by a failed read. No bounds check between `SDL_RWread()` return value and expected size.

### 2. **Iterator Validity Not Protected**
`cur_res_file_t` is a static iterator initialized only on first `open_res_file()`. Calling `cur_res_file()` before any file is opened dereferences an uninitialized/invalid iterator. No guard in `cur_res_file()`:
```cpp
SDL_RWops *cur_res_file(void) {
    res_file_t *r = *cur_res_file_t;  // DANGER: invalid iterator
    assert(r);
    return r->f;
}
```

### 3. **No Thread Safety**
Static `res_file_list` and `cur_res_file_t` are not guarded by mutexes. If two threads call `get_resource()` concurrently on different files, or one thread closes while another reads, undefined behavior occurs. Modern code would use `std::mutex` + scoped locks.

### 4. **Shallow Copy Constructor Risk**
`res_file_t` copy constructor copies the SDL_RWops pointer directlyΓÇöif two `res_file_t` instances exist for the same file, both will try to manage it, leading to double-close on destruction or use-after-free.
