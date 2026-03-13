# Source_Files/Misc/game_errors.cpp - Enhanced Analysis

## Architectural Role
This file implements a **global error sentinel** for the Aleph One engine, providing a simple single-error-at-a-time propagation mechanism that bridges subsystems (file I/O, map loading, network sync) with synchronization points (frame boundaries, level transitions). It's a callback-free alternative to exceptionsΓÇöappropriate for a 1990s cross-platform codebase that predates modern C++ exception adoption. The pattern allows **deferred error handling**: subsystems that detect failures can report them without blocking, while higher-level orchestrators (shell, game loop) check and clear at well-defined points.

## Key Cross-References

### Incoming (who depends on this file)
- **Files subsystem** (`game_wad.cpp`, `wad.cpp`, `resource_manager.cpp`): File I/O failures, WAD parsing errors, resource fork emulation problems ΓåÆ call `set_game_error()`
- **GameWorld subsystem** (`marathon2.cpp`, `map.cpp`): Physics/entity instantiation failures ΓåÆ call `set_game_error()` and `error_pending()`
- **Shell/Interface** (`interface.cpp`, `screen.cpp`): Main game loop orchestrator checks `error_pending()` at frame and level boundaries to detect deferred failures
- **Network subsystem** (`network_messages.cpp`, `game_wad.cpp`): Map checksum mismatches, sync failures ΓåÆ `set_game_error()`
- **XML/Lua subsystems**: Configuration/script parsing errors ΓåÆ `set_game_error()`

### Outgoing (what this file depends on)
- **cseries.h**: `assert()` macro for validation; no platform-specific calls
- **game_errors.h** (header): Error type/code enums (`systemError`, `gameError`, error code constants)
- **Standard C library**: No I/O, no allocations; purely in-process state

## Design Patterns & Rationale

**Global Error Sentinel Pattern**
- Single mutable `(last_type, last_error)` pair replaces exception throwing or per-function error codes
- **Why**: Pre-C++98 code; exceptions were expensive; no standard Result/Option types; decouples error reporting from immediate handling
- **Trade-off**: Fragile temporal coupling (error must be checked before overwritten); not thread-safe; unsuitable for concurrent error scenarios

**Sentinel Value (0 = no error)**
- `error_pending()` checks `last_error != 0`, treating 0 as default/cleared state
- **Why**: Allows cheap boolean check; avoids separate "has_error" flag
- **Trade-off**: Error code 0 is unusable; requires discipline in enum definition

**ScopedGameError RAII (header)**
- Mentioned in first-pass; allows temporary error suppression for non-critical code paths
- **Pattern**: Save-restore semantics for state that doesn't care about errors
- **Use case**: Optional asset loading that shouldn't abort level initialization if a skin fails

**DEBUG-only assertion on game error codes**
- `set_game_error(gameError, code)` asserts `code < NUMBER_OF_GAME_ERRORS` only in DEBUG builds
- **Why**: Catches invalid error codes early in development without production overhead
- **Trade-off**: Silently accepts invalid codes in Release builds

## Data Flow Through This File

```
Subsystems (Files, GameWorld, Network)
         Γåô
    set_game_error(type, code)
         Γåô
   [last_type, last_error] ΓåÉ static globals
         Γåô
   (checked passively by shell/game loop)
         Γåô
    error_pending() or get_game_error()
         Γåô
   clear_game_error() (at level/frame boundary)
```

**Key temporal constraint**: Error is only valid until next `set_game_error()` call or explicit `clear_game_error()`. No error queue; **last error wins** (overwrites previous). Shell code must poll between subsystem calls to avoid missing errors.

## Learning Notes

1. **Era-specific**: This pattern is characteristic of 1990sΓÇô2000s game engines before exception-safe code became standard. Modern engines use `Result<T, E>` (Rust-style) or typed exceptions.

2. **Determinism concern**: For a multiplayer game with replay validation (see cross-ref to `replay_film_test.cpp`), mutable global state is problematic. Replay code likely must save/restore error state or ensure errors don't affect simulation determinism.

3. **Type bifurcation**: Distinguishing `systemError` (OS-level) from `gameError` (logic-level) reflects separation of concerns, but the single-error storage means only one can be pending at a time.

4. **Thread-unsafety (subtle)**: The statics are not guarded by mutex. If file loading, network sync, or Lua callbacks run on background threads, concurrent `set_game_error()` calls corrupt state. Aleph One's architecture probably ensures serial write access via task scheduling (see `mytm_sdl.cpp`).

## Potential Issues

1. **Overwrite-without-check**: If subsystem A sets an error, then subsystem B sets another before A's error is checked, A's error is lost silently.

2. **No error codes for "no error"**: Error type 0 and code 0 are valid defaults but may conflict with legitimate enum values if not carefully defined.

3. **Thread-safety gap**: Background threads (music player, network I/O, asset cache) calling `set_game_error()` without synchronization risk data races on the statics. Likely mitigated by design (serial task queue), but not explicit in this code.

4. **Replay determinism**: Global mutable state may leak non-deterministic decisions (e.g., which file load failed first) into replay data if not carefully isolated.
