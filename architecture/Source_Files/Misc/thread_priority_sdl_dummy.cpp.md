# Source_Files/Misc/thread_priority_sdl_dummy.cpp

## File Purpose
Provides a dummy fallback implementation of thread priority boosting for SDL-based systems where platform-specific priority adjustments are unavailable. Logs a one-time warning and always succeeds gracefully to avoid breaking network code.

## Core Responsibilities
- Implement `BoostThreadPriority()` as a no-op stub with warning
- Suppress repeated warning messages using static state
- Maintain interface compatibility with platform-specific implementations

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope (global/static/singleton) | Purpose |
|------|------|------|---------|
| `didPrintOutWarning` | `bool` | static (function-local) | Tracks whether warning has been printed to avoid spam; initialized to `false`, set to `true` after first call |

## Key Functions / Methods

### BoostThreadPriority
- Signature: `bool BoostThreadPriority(SDL_Thread* inThread)`
- Purpose: Dummy implementation that satisfies the thread priority API contract on unsupported platforms.
- Inputs: `inThread` ΓÇö SDL thread pointer (ignored in this implementation).
- Outputs/Return: Always returns `true` (pretending success).
- Side effects (global state, I/O, alloc): Prints warning to stdout once via `printf()`; modifies static `didPrintOutWarning`.
- Calls (direct calls visible in this file): `printf()` (from `<stdio.h>`).
- Notes: Returns `true` to prevent caller from treating the operation as failed, even though actual priority boosting is not performed. The header comment indicates the intended behavior is to either boost the worker thread's priority or reduce the main thread's priority; this stub does neither but masks the limitation. Warning is printed at most once per process.

## Control Flow Notes
This file is a compile-time fallback: it is likely selected via conditional compilation or linker substitution when a platform-specific implementation (e.g., `thread_priority_sdl_posix.cpp`, `thread_priority_sdl_windows.cpp`) is not available. Called during engine initialization or network subsystem startup to attempt thread priority tuning.

## External Dependencies
- `SDL_Thread` (struct, forward-declared in `thread_priority_sdl.h`, from SDL library)
- `<stdio.h>` ΓåÆ `printf()`
