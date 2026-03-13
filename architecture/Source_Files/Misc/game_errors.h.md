# Source_Files/Misc/game_errors.h

## File Purpose
Defines the game engine's error handling system, including error type categories and error codes. Provides C-style functions for error state management and a C++ RAII class for temporary error suppression during specific operations.

## Core Responsibilities
- Define error type categories (system vs. game errors)
- Define game-specific error codes (file not found, index out of range, unsync, etc.)
- Provide functions to set, get, and clear global error state
- Query whether an error is pending
- Support scoped/automatic error state restoration via RAII pattern

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| (unnamed) | enum | Error type categories: `systemError`, `gameError` |
| (unnamed) | enum | Game error codes: `errNone`, `errMapFileNotSet`, `errIndexOutOfRange`, `errTooManyOpenFiles`, `errUnknownWadVersion`, `errWadIndexOutOfRange`, `errServerDied`, `errUnsyncOnLevelChange` |
| `ScopedGameError` | class | RAII wrapper to save/restore error state automatically on scope exit |

## Global / File-Static State
None (actual error state is managed elsewhere; this is a declarations-only header).

## Key Functions / Methods

### set_game_error
- Signature: `void set_game_error(short type, short error_code);`
- Purpose: Set the current game error state
- Inputs: `type` (error category), `error_code` (specific error identifier)
- Outputs/Return: void
- Side effects: Updates global error state
- Calls: Invoked by `ScopedGameError::~ScopedGameError()`
- Notes: Implementation defined elsewhere

### get_game_error
- Signature: `short get_game_error(short *type);`
- Purpose: Retrieve the current error and its type
- Inputs: `type` (output pointer for error category)
- Outputs/Return: error code (short); `*type` set to category
- Side effects: None
- Calls: Invoked by `ScopedGameError` constructor and client code
- Notes: Implementation defined elsewhere

### error_pending
- Signature: `bool error_pending(void);`
- Purpose: Check if any error is currently set
- Inputs: None
- Outputs/Return: `true` if error pending, `false` otherwise
- Side effects: None
- Calls: Implementation defined elsewhere
- Notes: Non-destructive query

### clear_game_error
- Signature: `void clear_game_error(void);`
- Purpose: Clear the current error (reset to no error state)
- Inputs: None
- Outputs/Return: void
- Side effects: Clears global error state
- Calls: Implementation defined elsewhere

### ScopedGameError (constructor/destructor)
- **Constructor**: Captures current error state on object creation
- **Destructor**: Restores previous error state when scope exits
- **Purpose**: RAII pattern for temporary error suppression; allows code to execute without polluting outer error state
- **Use case**: Wrap operations that may fail but failure is not critical

## Control Flow Notes
This is a global error registry system. The `ScopedGameError` class enables local error isolationΓÇöuseful for speculative operations or cleanup code that should not leave stale errors. The comment suggests the system would benefit from exception-based error handling in a modern refactor.

## External Dependencies
- C++ standard library (class syntax)
- Function implementations: not visible in this file (declared but defined elsewhere)
- No explicit `#include` directives shown
