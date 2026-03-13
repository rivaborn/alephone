# Source_Files/XML/XML_LevelScript.h - Enhanced Analysis

## Architectural Role

This header is the **level scripting orchestration gateway**ΓÇöit bridges level initialization from the game loop (GameWorld/marathon2.cpp) to the MML/Lua scripting engines (XML subsystem). It decouples *when* scripts execute (at level boundaries, game end) from *how* they're parsed, enabling designers to modify level behavior without engine recompilation. The dual MML/Lua support reveals a transitional architecture: MML for legacy Marathon content, Lua for extensibility.

## Key Cross-References

### Incoming (who calls these functions)
- **GameWorld/marathon2.cpp** (`update_world`, `begin_game`) ΓÇô Calls `RunLevelScript(level_index)` on level entry; likely calls `RunScriptChunks()` during frame updates if scripts support incremental execution
- **Misc/interface.cpp** (`begin_game`) ΓÇô Probably calls `LoadLevelScripts()` and `FindLevelMovie()` during game/level initialization
- **Files/game_wad.cpp** ΓÇô May call `SetMMLS()/SetLUAS()` when loading embedded scripts from map file resources
- **Rendering/Shell initialization** ΓÇô Calls `RunEndScript()` for end-game sequences (credits, etc.)

### Outgoing (subsystems called)
- **XML/XML_MakeRoot.cpp** ΓÇô Parses `InfoTree` configs via `parse_mml_default_levels()` (likely called during startup)
- **XML subsystem** (not explicitly visible) ΓÇô MML/Lua interpreter invoked during `RunLevelScript()`, `RunEndScript()`, `RunRestorationScript()`
- **Files/FileHandler.h** ΓÇô `FileSpecifier` used to locate and open map files and movie files
- **Lua subsystem** (Source_Files/lua/) ΓÇô `SetLUAS()/GetLUAS()` interface suggests Lua script embedding and execution
- **GameWorld (implicit)** ΓÇô Script execution modifies game parameters (entity spawning, platform states, difficulty)

## Design Patterns & Rationale

**Lifecycle-keyed script execution**: Scripts are cached once (`LoadLevelScripts`), then executed at specific engine states (`RunLevelScript`, `RunEndScript`, `RunRestorationScript`). This avoids re-parsing per-level and keeps script data in memory.

**Dual script storage (MML + Lua)**: Separate getter/setter pairs for MML and Lua suggest the engine evolved from pure MMLΓåÆ added Lua for richer scripting without breaking legacy content. Both are embedded in the map file's resource stream.

**Movie decoupling**: `FindLevelMovie()` and `GetLevelMovie()` are separateΓÇömovies are located during level setup, retrieved on-demand. This keeps the critical path fast if movies aren't played (e.g., skipped cinematics).

**Restoration as inversion**: `RunRestorationScript()` is the inverse of `RunLevelScript()`ΓÇöit resets parameters to defaults. This pattern suggests MML scripts have *side effects* on engine state that persist across level restarts or cleanup. Cleaner than tracking every modified parameter.

## Data Flow Through This File

```
Map File Resource 128
    Γåô
LoadLevelScripts() ΓåÆ cached in global state
    Γåô
On Level Entry:
  RunLevelScript(idx) ΓöÇΓåÆ MML/Lua Interpreter ΓöÇΓåÆ modifies game parameters
                                              ΓöÇΓåÆ spawns entities
  FindLevelMovie(idx) ΓöÇΓåÆ FileSpecifier cache
    Γåô
During Gameplay:
  RunScriptChunks() ΓöÇΓåÆ incremental script updates (if supported)
    Γåô
On Level Exit / Game End:
  RunRestorationScript() ΓöÇΓåÆ revert parameters to baseline
  RunEndScript() ΓöÇΓåÆ end-game MML (credits, state flags)
    Γåô
Movie Playback:
  GetLevelMovie() ΓöÇΓåÆ FileSpecifier* ΓöÇΓåÆ caller passes to show_movie()
```

## Learning Notes

- **Marathon Legacy**: Resource ID 128 for level scripts is Mac OS era (resource fork convention). Modern engines use file paths or embedded sections.
- **Dual Scripting Model**: MML (declarative markup for content) + Lua (imperative for logic) shows practical pragmatismΓÇödon't rewrite campaign content, add scripting layer on top.
- **Parameter Persistence**: The restoration script pattern reveals that MML *mutates* engine state deeply (difficulty, monster properties, weapon balances). This is higher-level than typical map metadata.
- **Separation of Concerns**: Script data (`SetMMLS/GetMMLS`) is distinct from execution (`RunLevelScript`)ΓÇöenables caching and late binding.

## Potential Issues

- **Ownership Semantics Unclear**: `SetMMLS(uint8* data, ...)` doesn't specify whether the function copies data or takes ownership. Callers may leak or double-free.
- **No Error Handling in Interface**: No return codes or exception declarations; callers can't detect parse/execution failures. Script errors may silently fail or crash the engine.
- **Movie Lookup Overhead**: `FindLevelMovie()` is called before `GetLevelMovie()`, but the header doesn't indicate whether it's cached or re-scanned each call. If re-scanned, per-frame calls could be expensive.
- **Global State Mutation**: `EndScreenIndex` and `NumEndScreens` are module globals. Multi-level games or replays could have state conflicts if not carefully sequenced during level transitions.
