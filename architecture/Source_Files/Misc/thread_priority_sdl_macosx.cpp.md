# Source_Files/Misc/thread_priority_sdl_macosx.cpp

## File Purpose
Provides macOS-specific thread priority elevation for SDL threads. Allows the engine to boost worker thread priorities to maximum for better performance on the target scheduling policy.

## Core Responsibilities
- Convert SDL thread IDs to POSIX pthread identifiers for macOS
- Query thread's current scheduling policy and parameters
- Set thread priority to the maximum allowed for that policy
- Return success/failure status to caller

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SDL_Thread` | opaque struct (external) | SDL abstraction of a thread |
| `pthread_t` | typedef (external) | POSIX thread identifier |
| `sched_param` | struct (external) | POSIX scheduling parameters |

## Global / File-Static State
None.

## Key Functions / Methods

### BoostThreadPriority
- **Signature:** `bool BoostThreadPriority(SDL_Thread* inThread)`
- **Purpose:** Elevate thread priority to maximum allowed by its scheduling policy
- **Inputs:** `inThread` ΓÇô SDL thread pointer to boost
- **Outputs/Return:** `bool` ΓÇô `true` if priority set successfully; `false` if `pthread_getschedparam()` or `pthread_setschedparam()` fails
- **Side effects:** Modifies thread's OS-level scheduling priority via POSIX APIs; succeeds silently or returns error code without fallback
- **Calls:**
  - `SDL_GetThreadID(inThread)` ΓÇô convert SDL_Thread to pthread_t
  - `pthread_getschedparam(theTargetThread, &theSchedulingPolicy, &theSchedulingParameters)` ΓÇô query policy and current priority
  - `sched_get_priority_max(theSchedulingPolicy)` ΓÇô lookup maximum priority for policy
  - `pthread_setschedparam(theTargetThread, theSchedulingPolicy, &theSchedulingParameters)` ΓÇô apply new priority
- **Notes:** Code comment states "this approach seems to work on Mac OS X (10.1.0, anyway)", suggesting platform-specific and version-dependent behavior. No error recovery: any API failure causes immediate return. The function modifies the target thread in-place without locking or synchronization primitives visible here.

## Control Flow Notes
Utility function for initialization: likely called by the main thread when spawning or initializing worker threads (e.g., rendering, sound). One-shot operation with no retry or fallback logic. No apparent place in typical frame/render loops; belongs to startup/shutdown or thread pool initialization.

## External Dependencies
- `<SDL2/SDL_thread.h>` ΓÇô SDL threading abstraction
- `<pthread.h>` ΓÇô POSIX threading APIs (getschedparam, setschedparam)
- `<sched.h>` ΓÇô Scheduling policy constants and priority query (sched_get_priority_max)
- `"thread_priority_sdl.h"` ΓÇô Header declaring `BoostThreadPriority()` function signature
