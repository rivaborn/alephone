# Source_Files/Files/tags.h - Enhanced Analysis

## Architectural Role

This file serves as the **semantic registry for the WAD binary format**, defining all chunk identifiers (tags) and file type classifications (typecodes) used throughout the engine. It sits at the boundary between game logic and binary serialization: files/wad.cpp reads tags to parse incoming WAD chunks, game_wad.cpp writes tags when serializing game state (player, monsters, platforms, Lua), and all GameWorld entity modules implicitly depend on consistent tag definitions. The typecode system further enables cross-platform portability by abstracting macOS resource fork type codes (`'snd '`, `'shp???'`) into logical file type enums, allowing the same code to work whether loading from Mac type codes or modern file extensions.

## Key Cross-References

### Incoming (who depends on this file)

- **Files subsystem**: wad.cpp uses tags to parse multi-version WAD formats; game_wad.cpp uses SAVE_* tags to serialize/deserialize entire game state; wad_prefs.cpp uses pref*_TAG constants for preferences storage
- **GameWorld serialization**: Player, Monster, Projectile, Platform, Terminal, and Effects structures use corresponding tags when saved (e.g., MONSTERS_STRUCTURE_TAG in game_wad.cpp)
- **Resource loading**: Sound system (tags for audio patches), Rendering (shape/image patches), Lua system (script tags MMLS_TAG, LUAS_TAG)
- **Physics system**: import_definitions.cpp uses MONSTER_PHYSICS_TAG, WEAPONS_PHYSICS_TAG, etc. to unpack physics models from both current format and M1_* legacy formats
- **Network subsystem**: map checksums and state sync rely on consistent tag identification

### Outgoing (what this file depends on)

- **cstypes.h**: Provides `uint32`, `FOUR_CHARS_TO_INT()` macro, `NONE` constant
- **filetypes_macintosh.c**: Contains actual OS type code mappings (e.g., `get_typecode(_typecode_scenario)` returns the 4-char code like `'scen'`); loaded at runtime via `initialize_typecodes()`
- **\<vector\>**: Included but unused (likely vestigial from older design or kept for future expansion)

## Design Patterns & Rationale

**1. Two-Level Indirection (Typecodes)**
- The Typecode enum abstracts platform-specific file type identifiers, decoupling game logic from macOS "creator/type" codes
- `get_typecode()`/`set_typecode()` functions allow runtime configuration, crucial because macOS type codes are not compile-time constants (loaded from resource fork)
- This pattern was necessary in 1991 when Marathon ran on 68K Macs; now enables portable builds where type codes are assigned at startup

**2. Chunk-Based WAD Registry**
- All 4-character tag constants enable extensible WAD format: new data types can be added by defining a new tag without breaking existing files
- Tags are compile-time constants (via FOUR_CHARS_TO_INT macro), enabling WAD readers to efficiently skip unknown chunks
- Supports multi-version WAD format (versions 0ΓÇô4) by allowing version-specific tags; old readers skip new tag types gracefully

**3. Legacy Compatibility Parallel (M1_* constants)**
- Marathon 1 physics tags (M1_MONSTER_PHYSICS_TAG, etc.) enable importing original Marathon scenarios; import_definitions.cpp attempts M1 format first, falls back to modern format
- This reflects the engine's evolution: Infinity compatibility became a design goal after Marathon 1 shipped

**4. Comments as Inline Changelog**
- The file's comment block (Feb 2000 ΓÇô Jul 2002) documents evolving compatibility decisions (sounds ΓåÆ `'snd???'`, shapes ΓåÆ `'shp???'`) showing iterative cross-platform support development

## Data Flow Through This File

**Incoming:**
1. **WAD file reading**: FileHandler opens .wad ΓåÆ wad.cpp calls `get_data_from_wad(tag)` ΓåÆ wad uses tag constants to locate chunks
2. **Game state serialization**: game_wad.cpp calls functions like `append_data_to_wad()` with PLAYER_STRUCTURE_TAG, MONSTERS_STRUCTURE_TAG, etc.
3. **Physics unpacking**: Extras/extract/ or import_definitions.cpp reads MONSTER_PHYSICS_TAG from .wad or .phys file

**Transformation:**
- Tags are used as dictionary keys in WAD chunk index (wad.cpp maintains tag ΓåÆ offset/size mapping)
- Typecodes translated at initialization via resource fork loader (macOS) or XML/INI defaults (other platforms)

**Outgoing:**
- Binary WAD on disk contains 4-byte tag headers; readers use tag constants to identify chunk types
- Typecodes returned by `get_typecode()` used in FileHandler to set OS file type metadata when saving scenarios, physics files, etc.

## Learning Notes

**What developers learn from this file:**
1. **WAD format structure**: The tag constants are a de facto schema specification; a developer can understand the WAD layout by reading this file
2. **Abstraction over platform**:Typecode indirection is a lesson in cross-platform design from the 90sΓÇô2000s era; modern engines use versioning schemes instead
3. **Iterative compatibility**: The comment history shows how compatibility decisions evolved (Marathon 1 ΓåÆ Infinity ΓåÆ Aleph One)
4. **Extensibility through tagging**: Rather than fixed binary format, tags allow new features (LUA_STATE_TAG, SAVE_META_TAG) to be added without breaking old files

**Idiomatic to this engine/era:**
- **4-char codes**: Direct legacy of macOS "OSType" convention (HyperCard XCMD, Mac Toolbox ResType)
- **Resource fork abstraction**: Reflects Marathon's origin as Mac-only game; resource_manager.cpp bridges AppleSingle/MacBinary formats
- **Hand-written binary serialization** (not protobuf, not msgpack): Typical of early-2000s open-source game ports
- **Static global typecode lookup**: No lazy loading or plugin architecture; all types known at compile time

## Potential Issues

1. **No collision detection**: If two developers independently add tags without coordinating, collisions are possible but would only manifest at runtime when parsing diverges
2. **Weak type safety**: Tag values are uint32 "magic numbers"; no compile-time enforcement that correct tag is used with correct structure (could write `MONSTERS_STRUCTURE_TAG` with wrong binary layout)
3. **Silent default fallback**: `get_typecode()` returns `'????'` for unrecognized types; if a typecode fails to load, the string placeholder hides the error
4. **Thread safety not documented**: `set_typecode()` modifies global state; unclear if safe to call during game runtime if multiple threads call `get_typecode()`
5. **Legacy format burden**: M1_* constants add maintenance burden; if M1 import becomes unused, they're dead code but hard to detect as such
