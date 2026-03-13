# Source_Files/Misc/game_errors.cpp

## File Purpose
Simple error tracking subsystem for the game engine. Maintains the last error code and type (system or game), providing a global error queue alternative for error state propagation during gameplay and initialization.

## Core Responsibilities
- Store and maintain the last error code and error type as static state
- Set error codes with type assertions for debugging (DEBUG mode)
- Retrieve the stored error code and optionally its type
- Check if a pending error exists without clearing it
- Clear error state for error handling cleanup

## Key Types / Data Structures
None (uses primitive `short` integers; enums defined in header).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `last_type` | `short` | static | Stores the type of the last error (systemError or gameError) |
| `last_error` | `short` | static | Stores the error code of the last error (0 = no error) |

## Key Functions / Methods

### set_game_error
- **Signature:** `void set_game_error(short type, short error_code)`
- **Purpose:** Set the current error state with a type and code.
- **Inputs:** `type` (error type: systemError or gameError), `error_code` (enumerated error code)
- **Outputs/Return:** None
- **Side effects:** Modifies `last_type` and `last_error` statics; asserts on invalid type in DEBUG builds
- **Calls:** `assert()` (C stdlib)
- **Notes:** Validates `type >= 0 && type < NUMBER_OF_TYPES` unconditionally; in DEBUG mode, also validates game error codes. Overwrites any previous error state.

### get_game_error
- **Signature:** `short get_game_error(short *type)`
- **Purpose:** Retrieve the stored error code and optionally its type.
- **Inputs:** `type` (optional pointer to receive the error type; can be null)
- **Outputs/Return:** Returns the error code; writes type to `*type` if pointer is non-null
- **Side effects:** None
- **Calls:** None
- **Notes:** Non-destructive read; does not clear error state.

### error_pending
- **Signature:** `bool error_pending(void)`
- **Purpose:** Check if an error has been set.
- **Inputs:** None
- **Outputs/Return:** Boolean true if `last_error != 0`
- **Side effects:** None
- **Calls:** None
- **Notes:** Simple sentinel check; treats error code 0 as "no error".

### clear_game_error
- **Signature:** `void clear_game_error(void)`
- **Purpose:** Reset error state to cleared.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets both `last_error` and `last_type` to 0
- **Calls:** None
- **Notes:** Unconditionally clears; no checks for pending errors.

## Control Flow Notes
This is a utility module that operates passively. Callers in the engine set errors via `set_game_error()` during critical operations (file loading, WAD indexing, multiplayer sync); other code checks with `error_pending()` or `get_game_error()` at synchronization points (frame boundaries, level transitions). The `ScopedGameError` RAII class (defined in header) allows temporary error state savings for code paths that don't care about errors.

## External Dependencies
- **cseries.h** ΓÇô platform abstraction, assert(), fundamental types
- **game_errors.h** ΓÇô error type and code enums, function declarations
- None other (no stdlib beyond assert, no I/O)
