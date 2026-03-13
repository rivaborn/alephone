# Source_Files/XML/XML_LevelScript.cpp - Enhanced Analysis

## Architectural Role

This file is the **execution orchestrator** for per-level configuration and scripting in Aleph One. It acts as a coordinator hub that deserializes XML-based level scripts from map files, maintains a per-level registry, and dispatches execution to specialized subsystems (MML parser, Lua engine, music player, OpenGL load screens). The three-tier execution model (Default ΓåÆ Level-specific ΓåÆ Pseudo-levels) allows level designers to override global defaults while the engine can inject restoration/end-of-game hooks without code branching.

## Key Cross-References

### Incoming (who depends on this)
- **Misc/interface.cpp** (`begin_game`): Calls `LoadLevelScripts()` on map load to populate the registry from resource 128
- **GameWorld/marathon2.cpp** (main game loop): Calls `RunLevelScript()` before each level and `RunEndScript()` at game end
- **Game state management** (via shell.h): Calls `ResetLevelScript()` to fade music and reset MML state between levels
- **Movie playback** (show_movie): Calls `FindLevelMovie()` then `GetLevelMovie()` to retrieve cached movie data
- **Files/game_wad.cpp**: Via `SetMMLS()`/`SetLUAS()` to inject binary script chunks from decompressed WAD data

### Outgoing (what this file depends on)
- **XML_ParseTreeRoot.h**: `ParseMMLFromData()` parses MML text into the engine state
- **lua_script.h**: `LoadLuaScript()` loads and executes Lua scripts with `_embedded_lua_script` context
- **Music.h** (subsystem): `Music::instance()->Fade()`, `PushBackLevelMusic()`, `SetPlaylistParameters()`, `SeedLevelMusic()`
- **OGL_LoadScreen.h** (conditional): `OGL_LoadScreen::instance()->Set()` configures load screen appearance and progress colors
- **Plugins.h**: `Plugins::instance()->load_mml()` applies plugin-provided MML on level reset
- **InfoTree.h**: XML DOM parsing, node iteration, attribute extraction
- **images.h**: `get_text_resource_from_scenario()` retrieves TEXT resource 128 (map script)
- **AStream.h**: `AIStreamBE` for big-endian binary parsing of mmls_chunk / luas_chunk
- **Logging.h**: `logError()` for XML parse failure reporting

## Design Patterns & Rationale

**Registry + Pointer Caching**: `LevelScripts` map + `CurrScriptPtr` avoid repeated lookups during script execution. The cached pointer is set once per `GeneralRunScript()` call and used for the entire command loopΓÇöa micro-optimization valuable when iterating over potentially dozens of commands.

**Three-Tier Fallback**: Default ΓåÆ Level-specific ΓåÆ Pseudo-levels creates a composable configuration model. Default settings are inherited by all levels; pseudo-levels (-2, -1, -3) reuse the same execution pipeline for special purposes without code duplication.

**Dual-Source Loading**: XML (resource 128) for human-editable map scripts + binary chunks (mmls_chunk, luas_chunk) for internal WAD format. This decouples map distribution (text-friendly XML) from in-memory representation (binary chunks for compression/efficiency).

**Dispatch by Command Type**: The switch statement in `GeneralRunScript()` dispatches to subsystem-specific handlers (MML parser, Lua loader, music player, load screen) with unified error handling. This inverts controlΓÇöthe script engine doesn't know MML syntax; it delegates.

**State Separation**: Movie metadata (MovieFile, MovieFileExists, MovieSize) is cached separately from command execution, suggesting movies are resolved and cached before playback to avoid file I/O during critical rendering.

## Data Flow Through This File

```
MAP FILE
  ΓööΓöÇ Resource 128 (TEXT)
       ΓööΓöÇ XML <marathon_levels>
            Γö£ΓöÇ parse_levels_xml()
            Γöé   ΓööΓöÇ parse_level_commands() per <level>/<default>/<end>/<restore>
            Γöé       ΓööΓöÇ populate LevelScripts[index].Commands[]
            ΓööΓöÇ on load error
                 ΓööΓöÇ logError()

EXECUTION PATH (per level):
1. RunLevelScript(N)
   Γö£ΓöÇ GeneralRunScript(Default)  [MML, Music, Lua, LoadScreen]
   Γö£ΓöÇ GeneralRunScript(N)
   ΓööΓöÇ Music::SeedLevelMusic()

2. GeneralRunScript(LevelIndex)
   Γö£ΓöÇ lookup CurrScriptPtr = &LevelScripts[LevelIndex]
   Γö£ΓöÇ Music::SetPlaylistParameters()
   ΓööΓöÇ for each Command:
      Γö£ΓöÇ MML ΓåÆ ParseMMLFromData()
      Γö£ΓöÇ Lua ΓåÆ LoadLuaScript()
      Γö£ΓöÇ Music ΓåÆ Music::PushBackLevelMusic()
      Γö£ΓöÇ LoadScreen ΓåÆ OGL_LoadScreen::Set()
      ΓööΓöÇ Movie ΓåÆ cache in MovieFile/MovieFileExists/MovieSize

MOVIE RETRIEVAL:
  FindLevelMovie(index)
  Γö£ΓöÇ FindMovieInScript(Default)
  Γö£ΓöÇ FindMovieInScript(index)
  ΓööΓöÇ GetLevelMovie() ΓåÆ returns cached MovieFile + Size

BINARY CHUNK EXECUTION:
  mmls_chunk[] / luas_chunk[]
  Γö£ΓöÇ RunScriptChunks()
  Γöé   Γö£ΓöÇ AIStreamBE parses big-endian headers (flags, name, length)
  Γöé   Γö£ΓöÇ bounds-check before access
  Γöé   ΓööΓöÇ dispatch ParseMMLFromData() / LoadLuaScript()
  ΓööΓöÇ (called by game_wad.cpp after decompression)
```

## Learning Notes

**Pseudo-Level Indexing**: Using negative indices (Restore=-3, Default=-2, End=-1) to distinguish special pseudo-levels from regular level indices (0+) is a clever minimalist designΓÇöno separate enum or state machine needed, just conditional logic on the index sign.

**FirstTime Flag Quirk**: The `static bool FirstTime` in `LoadLevelScripts()` preserves external level scripts on initial load but clears the map on subsequent loads. This implies plugins can register level scripts before any map is loaded, and those survive the first `LoadLevelScripts()` call. Unusual but intentional.

**OpenGL Gating**: Load screen setup is `#ifdef HAVE_OPENGL`, showing the engine supports non-OpenGL rendering modes (e.g., software rasterizer) where load screens don't apply.

**Big-Endian Binary Format**: `RunScriptChunks()` uses `AIStreamBE` to parse mmls_chunk and luas_chunk in big-endian format. This suggests the binary format was designed for PowerPC Mac compatibility (even though the engine now runs on x86/ARM via SDL).

**Music as Deferred State**: Music files aren't played immediately; they're pushed to a playlist via `PushBackLevelMusic()` and sequenced by `SeedLevelMusic()` during level transitions. This decouples level script parsing from playback timing.

## Potential Issues

1. **Static Global Lifetime**: `LevelScripts`, `CurrScriptPtr`, `MovieFile`, `mmls_chunk`, `luas_chunk` are file-static globals with implicit lifecycle tied to module initialization order. If another subsystem accesses these before `LoadLevelScripts()` is called, behavior is undefined.

2. **Unprotected Concurrent Access**: `CurrScriptPtr` is a raw pointer to a map element. If `LevelScripts` is modified (e.g., in a reload scenario) while `GeneralRunScript()` is iterating, the pointer could dangle.

3. **ResourceID Caching**: Once a resource ID is loaded into `ScriptRsrc` via `get_text_resource_from_scenario()`, subsequent reads of the same RsrcID within the same level don't re-fetch from disk. If the resource changes mid-execution, stale data persists. (Minor issue since maps are typically immutable during gameplay.)

4. **Bounds Checking in Binary Parsing**: `RunScriptChunks()` validates `offset + 8 + LEVEL_NAME_LENGTH > chunk.size()` but later reads the full `length` field. If `length` is maliciously large, `offset + length > chunk.size()` could silently truncate the script. The `break` statement halts processing but doesn't log a warning.
