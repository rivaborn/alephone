# Source_Files/Misc/game_errors.h - Enhanced Analysis

## Architectural Role

`game_errors.h` is a **cross-cutting infrastructure layer** providing global error state management across the entire engine. Nearly every major subsystem (Files, GameWorld, Network, Rendering) writes errors to this registry when failures occur, while the UI/Shell layer queries it to display error dialogs to the user. The file's simplicity and declarations-only structure reflects its role as a minimal-overhead foundation that bridges low-level subsystems (file I/O, network, entity simulation) with high-level UI concerns.

## Key Cross-References

### Incoming (who depends on this file)

**Error setters** (subsystems that write errors):
- **Files subsystem** sets `errMapFileNotSet`, `errUnknownWadVersion`, `errWadIndexOutOfRange`, `errTooManyOpenFiles` when file operations fail during map/WAD loading
- **Network subsystem** sets `errServerDied` when peer-to-peer connections drop or metaserver communication fails
- **GameWorld subsystem** sets `errUnsyncOnLevelChange` during multiplayer level transitions if client/server state diverges
- **Platform/system code** sets `errIndexOutOfRange` for bounds violations in resource access

**Error getters** (subsystems that read errors):
- **UI/Shell layer** (`interface.cpp`, `shell.cpp`) calls `error_pending()` and `get_game_error()` to detect failures and display error dialogs
- **Main game loop** (`marathon2.cpp`) checks for errors during world updates and handles level transition failures
- **Lua scripting** may query error state to implement error-handling callbacks

### Outgoing (what this file depends on)

- **No explicit dependencies visible**: This is a declarations-only header with no `#include` directives
- **Function implementations** likely defined in `Source_Files/Misc/game_errors.cpp` (not shown), which manages the actual global error state (possibly a static struct or thread-local storage)
- **C++ standard library** (for `class` syntax only; no STL used)

## Design Patterns & Rationale

**Global Error Registry Pattern** (C-style):
- Mimics Unix `errno` model: set errors globally, query asynchronously
- Chosen for **minimal overhead** (important for a real-time game engine where performance is critical)
- Allows error propagation through call stacks without changing function signatures
- Works in C; easy to port from Marathon 1/2 legacy code

**RAII Wrapper** (`ScopedGameError`):
- Modern C++ improvement on the C pattern
- **Rationale**: Cleanup code (e.g., temporary texture loading, speculative pathfinding) may fail internally without propagating to the caller
- Saves/restores error state on scope exit, ensuring outer code doesn't see stale errors
- Developer comment acknowledges the system "badly needs fixing (suggestion: use exceptions)" ΓÇö this class is a pragmatic workaround, not the intended final design

**Error Type Categorization**:
- `systemError` vs `gameError` suggests layered error handling (OS-level failures vs. game logic failures)
- Allows UI to display different dialogs or recovery strategies based on error origin

## Data Flow Through This File

1. **Error generation**: Subsystem detects failure ΓåÆ calls `set_game_error(type, code)`
2. **Error querying**: UI layer calls `error_pending()` to check if error exists, then `get_game_error(&type)` to retrieve code and type
3. **Error clearing**: After displaying/handling error, caller invokes `clear_game_error()` to reset state
4. **Scoped isolation**: Optional wrapping with `ScopedGameError()` to suppress error pollution for non-critical operations

Key state transitions:
- **No error** ΓåÆ `set_game_error()` ΓåÆ **Error pending** ΓåÆ `get_game_error()` ΓåÆ **Error pending** ΓåÆ `clear_game_error()` ΓåÆ **No error**
- **ScopedGameError** ctor saves state on entry; dtor restores previous state on exit (nesting-safe)

## Learning Notes

**What developers learn from this file:**
1. **Deterministic error handling in real-time engines**: Global error state avoids recursive function signatures but requires disciplined clearing
2. **C-to-C++ migration pragmatism**: RAII wrapper retrofitted onto C patterns rather than full exceptions (reflects transition era of game engine development, ~1990sΓåÆ2010s)
3. **Cross-subsystem communication**: Errors are one of the few global state channels; most other communication is explicit parameter passing or callbacks
4. **Legacy Marathon compatibility**: Error codes (`errUnknownWadVersion`, `errWadIndexOutOfRange`) are likely inherited from Marathon 1/2 for save-game/map compatibility

**What's idiomatic to this engine/era:**
- Enumerated error codes instead of exception types (no dynamic error objects)
- Manual polling (`error_pending()`) instead of try/catch
- Global mutable state instead of error return values or Result<T> types
- No use of `std::optional`, `std::variant`, or modern C++ error handling

## Potential Issues

1. **Thread safety**: No synchronization visible; concurrent subsystem updates (network thread, rendering thread, main game loop) could race on global error state. The comment about "badly needs fixing" may partly refer to this architectural limitation.
2. **Error overwriting**: If one subsystem sets an error while another is still handling a previous error, the old error is lost (no error queue/history).
3. **Scoped nesting not immune to bugs**: If `ScopedGameError` is misused (exception during dtor, or exception in ctor before state capture), restoration could be incorrect.
