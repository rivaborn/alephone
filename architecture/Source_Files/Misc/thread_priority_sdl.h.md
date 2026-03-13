# Source_Files/Misc/thread_priority_sdl.h

## File Purpose
Header file that declares a utility function for boosting thread priorities in an SDL-based game engine. Provides a platform-abstraction interface for prioritizing performance-critical threads (likely rendering or game update loops) while reducing main thread priority if necessary.

## Core Responsibilities
- Declare the thread priority boosting function
- Ensure cross-platform thread priority management via SDL
- Allow callers to prioritize specific threads without platform-specific code

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SDL_Thread | opaque struct (forward-declared) | SDL thread handle; definition in SDL library |

## Global / File-Static State
None.

## Key Functions / Methods

### BoostThreadPriority
- Signature: `bool BoostThreadPriority(SDL_Thread* inThread)`
- Purpose: Elevate the priority of a specified thread, or if that fails, reduce the main thread's priority to ensure the target thread gets better scheduling.
- Inputs: `inThread` ΓÇö pointer to an SDL_Thread to be prioritized
- Outputs/Return: `bool` ΓÇö success status (true if priority adjustment succeeded)
- Side effects: Modifies OS-level thread scheduling priorities; main thread priority should not be further reduced by subsequent calls
- Calls: Not inferable from this file (implementation in .c file, likely uses OS APIs and SDL functions)
- Notes: Must be called by the main thread to allow main-thread priority reduction; assumes idempotent or carefully-managed priority state to avoid cascading reductions

## Control Flow Notes
Utility function for initialization/setup phase. Called when engine creates or starts performance-critical threads (e.g., rendering, physics update) to ensure they are not starved by the main thread or other OS tasks.

## External Dependencies
- SDL (Simple DirectMedia Layer): `SDL_Thread` opaque type from `<SDL.h>` or equivalent
