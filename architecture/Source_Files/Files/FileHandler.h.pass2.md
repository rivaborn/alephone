# Source_Files/Files/FileHandler.h - Enhanced Analysis

## Architectural Role

FileHandler.h is the **central I/O abstraction layer** for all file, directory, and resource access in Aleph One. It isolates the entire engine from SDL_RWops (and thus from OS-level file APIs) while also providing macOS resource fork emulation across all platforms. Every subsystem that loads maps, saves games, reads textures, reads sounds, loads Lua scripts, or persists state must route through these abstractions. This makes FileHandler.h the critical integration point between the game engine's logical entity model (worlds, entities, assets) and the persistent storage layer.

## Key Cross-References

### Incoming (who depends on this file)
- **GameWorld subsystem** (`game_wad.cpp`, `map.cpp`) ΓÇö Loads map data, save games, physics definitions via `FileSpecifier::Open()` and binary stream reading
- **Rendering subsystem** (`shapes.cpp`, `OGL_Textures.cpp`, `ImageLoader_SDL.cpp`) ΓÇö Loads shape collections, textures, and image caches via `FileSpecifier` for asset discovery
- **Audio subsystem** (`Music.cpp`, `SndfileDecoder.cpp`) ΓÇö Loads sound files and music via `FileSpecifier::Open()` and resource fork queries
- **Lua/XML subsystem** (`XML_MakeRoot.cpp`) ΓÇö Resolves script and configuration file paths via `SetNameWithPath()` and `SetToPreferencesDir()`
- **Preferences/Misc** (`preferences.cpp`, `achievements.cpp`, `wad_prefs.cpp`) ΓÇö Serializes user settings and progress via `FileSpecifier` directory helpers
- **Network subsystem** (`network_games.cpp`) ΓÇö Maps checksums and scenario lookups via WAD file introspection
- **Videos subsystem** (`Movie.cpp`) ΓÇö Records and plays back film data via `FileSpecifier` and `OpenedFile`

### Outgoing (what this file depends on)
- **SDL2 (`<SDL2/SDL.h>`)** ΓÇö All actual I/O via `SDL_RWops` handle lifecycle (open, read, write, seek, close)
- **Boost.IOStreams** ΓÇö Generic device adapter interface for wrapping `OpenedFile` as a C++ stream (`opened_file_device`)
- **tags.h** ΓÇö Typecode constants (`_typecode_creator`, file type enumerations) for cross-platform file type metadata
- **CSeries (implicit)** ΓÇö Likely uses `cspaths_sdl.cpp` functions to resolve per-user data directories (`SetToLocalDataDir()`, etc.)

## Design Patterns & Rationale

**RAII for Resource Lifecycle**  
All three resource classes (`OpenedFile`, `LoadedResource`, `OpenedResourceFile`) auto-close/unload in their destructors. This was considered modern for ~2000 code and prevents handle leaks. The design trades explicit error checking (which must happen via `GetError()`) for guaranteed cleanup.

**Adapter Pattern (opened_file_device)**  
Wraps `OpenedFile` to conform to Boost.IOStreams' `seekable_device_tag`, allowing legacy C++98 code to read binary files via operator overloading (`>>`) rather than manual buffer management. This bridges the low-level SDL_RWops layer to higher-level abstractions like `BIStream` / `AIStream`.

**Path Resolution with Search Paths**  
`SetNameWithPath()` implements a **asset directory search** pattern common in game engines: given a relative path like `"Sounds/Weapons/Rifle"`, it searches the data search path (typically: installed data dirs, mod directories, Steam Workshop) and returns the first match. `ScopedSearchPath` uses RAII to temporarily prepend a directory to this search list, matching the idiom of old Macintosh FSSpec resolution.

**Resource Fork Abstraction**  
`OpenedResourceFile` / `LoadedResource` abstract away Marathon's original macOS resource fork storage. On Mac, resources lived in the file's fork; on cross-platform, the engine emulates forks by:
- Detecting resource file formats (AppleSingle, MacBinary) via `resource_manager.cpp`
- Parsing 4-character type codes into resource IDs
- Returning malloc'd memory (`LoadedResource::SetData()`) rather than locked handles

This allows modern non-Mac ports to work with legacy asset formats.

**Separation of Path, File Handle, and Memory**  
Three independent concerns:
- **FileSpecifier** ΓÇö Path resolution, metadata queries, file operations (exists, delete, rename)
- **OpenedFile** ΓÇö Wraps active handle; position/length queries; read/write operations
- **LoadedResource** ΓÇö Holds malloc'd blocks; auto-deallocation; optional ownership transfer via `GetPointer(true)`

This separation allows code to reason about paths without open handles, avoiding resource leaks and handle exhaustion.

## Data Flow Through This File

**File Load Flow:**
```
FileSpecifier ΓåÆ SetNameWithPath(relative_path)  [searches data directories]
                Γåô
              Exists(), IsDir(), GetType()      [validates and identifies]
                Γåô
              Open(OpenedFile&)                 [acquires SDL_RWops handle]
                Γåô
        Read(count, buffer) [repeat]            [extracts raw bytes]
                Γåô
              ~OpenedFile()                      [auto-closes handle]
```

**Resource Fork Load Flow (macOS-compatible):**
```
FileSpecifier ΓåÆ Open(OpenedResourceFile&, writable=false)  [opens or emulates fork]
                Γåô
              Push()                             [saves prior fork context]
                Γåô
              Check(type, id) or Get(type, id, LoadedResource&)  [queries/loads]
                Γåô
              LoadedResource ΓåÆ SetData(ptr, len)  [adopts malloc'd memory]
                Γåô
              Pop()                              [restores prior fork]
                Γåô
              ~LoadedResource()                  [auto-frees memory]
```

**Directory Search and Discovery:**
```
FileSpecifier(relative_path) ΓåÆ canonicalize_path()  [normalize separators]
                Γåô
              SetNameWithPath()                     [search data dirs; return first hit]
                Γåô
              ReadDirectory() / ReadZIP()           [enumerate contents as vector<dir_entry>]
```

**Preference Persistence:**
```
FileSpecifier ΓåÆ SetToPreferencesDir()              [point to per-user prefs location]
                Γåô
              Open(OpenedFile&, Writable=true)     [create/overwrite]
                Γåô
              Write(count, buffer) [repeat]        [serialize settings]
                Γåô
              ~OpenedFile() [auto-close]
```

## Learning Notes

**Modern Engines Differ:**
1. **Direct Path Strings** ΓÇö Modern engines (Unity, Unreal) often use `string` paths directly; Aleph One separates path objects (`FileSpecifier`) from I/O operations, requiring explicit `Open()` calls. This is more type-safe but verbose.
2. **No Resource Forks** ΓÇö Modern engines abandoned resource fork abstractions entirely; Aleph One's emulation layer (`OpenedResourceFile`, `LoadedResource`) exists solely for legacy asset compatibility. Modern code would use flat asset directories with explicit type metadata (file extensions, YAML descriptors).
3. **Asset Streams vs. Malloc** ΓÇö Modern C++ would use `std::unique_ptr<char[]>` or `std::vector<char>` for loaded data. Aleph One's `LoadedResource` holds raw pointers and requires manual `SetData()` / `Unload()` calls, relying on RAII for cleanup but not type safety.
4. **Search Paths as Initialization** ΓÇö `ScopedSearchPath` predates dependency injection; a modern engine would configure search paths at startup, not modify global state at runtime. However, this design does enable clean mod loading and layered asset overrides.
5. **Platform Dialog Integration** ΓÇö `ReadDialog()`, `WriteDialog()`, `WriteDialogAsync()` are platform-specific UI calls, suggesting Aleph One runs on Desktop OSes (macOS, Windows, Linux) with native file pickers. Mobile/console engines typically use custom UI.

**What's Idiomatic to Early-2000s Game Dev:**
- Heavy use of typecodes (4-char constants from `tags.h`) rather than file extensionsΓÇöreflects Marathon's origin on systems without standardized extensions
- Manual error checking via `GetError()` rather than exceptionsΓÇöC++ exceptions were still considered too expensive for game code
- Binary blob serialization (`AStream`, `BStream`) for save game dataΓÇöthis predates JSON/Protocol Buffers

## Potential Issues

1. **Error Handling Opacity** ΓÇö `GetError()` returns a platform-specific error code (`-1` for unknown). Callers have no semantic way to distinguish "file not found" from "permission denied." An error enum would be more robust.

2. **Resource Fork Stack Corruption Risk** ΓÇö `OpenedResourceFile::Push()` / `Pop()` maintain a global resource fork stack. If an exception occurs between `Push()` and `Pop()`, the stack becomes corrupted. Code relies on discipline; modern C++ would use RAII guards.

3. **Buffer Overrun Risk in Read/Write** ΓÇö `OpenedFile::Read(count, buffer)` and `Write(count, buffer)` require the caller to guarantee `buffer` is large enough for `count` bytes. No bounds checking.

4. **Path Canonicalization Fragility** ΓÇö `canonicalize_path()` is private and called only at construction. If a `FileSpecifier` is constructed with a path and later modified via `AddPart()`, intermediate states may not be canonical, leading to duplicate lookups in search paths.

5. **Synchronous Dialog Blocking** ΓÇö `ReadDialog()` and `WriteDialog()` block; only `WriteDialogAsync()` allows background work. This can freeze game logic during file selection, though `WriteDialogAsync` mitigates this for saves.
