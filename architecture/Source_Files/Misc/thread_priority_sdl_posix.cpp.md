# Source_Files/Misc/thread_priority_sdl_posix.cpp

## File Purpose
Implements thread priority boosting for SDL threads on POSIX systems (Linux/Unix/macOS). Provides a platform-specific wrapper that elevates a thread to maximum scheduling priority within its current scheduling policy.

## Core Responsibilities
- Boost SDL thread priority to the maximum allowed by the system scheduler
- Retrieve and modify thread scheduling policy and parameters using POSIX APIs
- Conditionally compile for systems supporting POSIX priority scheduling
- Return success/failure status to caller

## Key Types / Data Structures
None (uses external types only: SDL_Thread, pthread_t, struct sched_param).

## Global / File-Static State
None.

## Key Functions / Methods

### BoostThreadPriority
- **Signature:** `bool BoostThreadPriority(SDL_Thread* inThread)`
- **Purpose:** Elevate the specified SDL thread to maximum scheduling priority on POSIX systems.
- **Inputs:** `inThread` ΓÇô pointer to an SDL_Thread object
- **Outputs/Return:** `bool` ΓÇô `true` on success (or on non-POSIX systems), `false` if scheduling operations fail
- **Side effects:** Modifies the target thread's scheduler priority via `pthread_setschedparam()`; no global state modified
- **Calls (direct):**
  - `SDL_GetThreadID(inThread)` ΓÇô extract native pthread_t from SDL_Thread wrapper
  - `pthread_getschedparam(theTargetThread, &theSchedulingPolicy, &theSchedulingParameters)` ΓÇô query current policy/params
  - `sched_get_priority_max(theSchedulingPolicy)` ΓÇô retrieve max priority for the policy
  - `pthread_setschedparam(theTargetThread, theSchedulingPolicy, &theSchedulingParameters)` ΓÇô apply new priority
- **Notes:**
  - Guarded by `#if defined(_POSIX_PRIORITY_SCHEDULING)` preprocessor check; returns `true` unconditionally on non-POSIX platforms
  - Assumes the calling thread has permission to modify the target thread's priority (typically requires elevated privileges)
  - Preserves the existing scheduling policy; only modifies the priority parameter within that policy

## Control Flow Notes
Called by the main thread to elevate priority of a game logic or audio thread. Per the header comment, this function is intended as a workaround when the full priority-boosting design (main thread deprioritization + worker thread boost) cannot be fully implemented. Returns immediately; does not loop or yield.

## External Dependencies
- `SDL2/SDL_thread.h` ΓÇô SDL threading abstractions (SDL_Thread, SDL_GetThreadID)
- `pthread.h` ΓÇô POSIX thread API (pthread_t, pthread_getschedparam, pthread_setschedparam)
- `sched.h` ΓÇô POSIX scheduler API (sched_get_priority_max, struct sched_param)
- `thread_priority_sdl.h` ΓÇô local module header (function declaration)
