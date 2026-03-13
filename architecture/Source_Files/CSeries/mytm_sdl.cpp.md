# Source_Files/CSeries/mytm_sdl.cpp

## File Purpose
Implements an SDL-based emulation of Apple's Time Manager API for periodic task scheduling. Provides drift-free timer callbacks for networking code and other subsystems that require scheduled execution. Tasks run in dedicated threads with mutual exclusion to prevent concurrent execution.

## Core Responsibilities
- Initialize and maintain a global mutex protecting timer task execution
- Create periodic timer tasks that execute in independent SDL threads
- Implement drift-free period scheduling (compensate for execution delays)
- Enforce mutual exclusion between timer tasks
- Clean up completed tasks and reclaim thread/memory resources
- Profile task execution timing and deadline misses (DEBUG builds only)
- Boost thread priority of timer tasks relative to main thread

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `myTMTask` | struct | Housekeeping structure for a single timer task: thread handle, period, callback, keep-running flag, profiling data |
| `myTMTask_profile` | struct | DEBUG only; tracks timing statistics: start/end time, call counts, drift range, late call count, warm resets |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sTMTaskMutex` | `SDL_mutex*` | static | Protects exclusive execution of timer task callbacks |
| `sOutstandingTasks` | `vector<myTMTaskPtr>` | static | Tracks all active timer tasks for cleanup |

## Key Functions / Methods

### mytm_initialize()
- **Signature:** `void mytm_initialize()`
- **Purpose:** One-time initialization; creates the global timer task mutex.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates `sTMTaskMutex` on first call; warns if called multiple times.
- **Calls:** `SDL_CreateMutex()`, `logWarning()`, `logAnomaly()`
- **Notes:** No cleanup function exists; mutex is leaked until process exit.

### thread_loop()
- **Signature:** `static int thread_loop(void* inData)` (SDL thread entry point)
- **Purpose:** Main loop executed by each timer task thread; implements drift-free periodic scheduling.
- **Inputs:** `inData` ΓÇô pointer to `myTMTask` structure
- **Outputs/Return:** Always returns 0
- **Side effects:** Calls user callback repeatedly; modifies `mKeepRunning` flag; locks/unlocks mutex; updates profiling data.
- **Calls:** `machine_tick_count()`, `sleep_for_machine_ticks()`, `take_mytm_mutex()`, `release_mytm_mutex()`, user-supplied `mFunction()`
- **Notes:** 
  - Computes delay as `(period - drift)` to compensate for timing errors.
  - If delay Γëñ 0, a deadline was missed; increments late-call counter.
  - Drift can drift across calls (see history comment: originally uninitialized signed arithmetic caused very long waits).
  - Minor known bug: function can be called once after `myTMRemove()` returns due to lack of final synchronization.

### myXTMSetup()
- **Signature:** `myTMTaskPtr myXTMSetup(int32 time, bool (*func)(void))`
- **Purpose:** Create a new periodic timer task.
- **Inputs:** `time` ΓÇô period in machine ticks; `func` ΓÇô callback returning `true` to reschedule, `false` to stop.
- **Outputs/Return:** Pointer to newly allocated `myTMTask`; added to `sOutstandingTasks`.
- **Side effects:** Allocates new task structure, creates SDL thread, boosts thread priority, stores in vector.
- **Calls:** `new`, `SDL_CreateThread()`, `BoostThreadPriority()`, `sOutstandingTasks.push_back()`
- **Notes:** Callback should not block; thread priority is higher than main thread to prevent starvation.

### myTMRemove()
- **Signature:** `myTMTaskPtr myTMRemove(myTMTaskPtr task)`
- **Purpose:** Stop a running timer task.
- **Inputs:** `task` ΓÇô pointer to task to stop (or NULL).
- **Outputs/Return:** Always returns NULL.
- **Side effects:** Sets `task->mKeepRunning = false`; does not wait for thread to exit.
- **Calls:** None
- **Notes:** Non-blocking; actual thread cleanup deferred to `myTMCleanup()`. Small window where callback may execute after this returns.

### myTMCleanup()
- **Signature:** `void myTMCleanup()`
- **Purpose:** Reap finished timer tasks, reclaim memory and thread handles.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Iterates `sOutstandingTasks`; removes and deletes stopped tasks; dumps profiling data (DEBUG); waits for threads.
- **Calls:** `sOutstandingTasks.erase()`, `myTMDumpProfile()`, `SDL_WaitThread()`, `delete`
- **Notes:** Called periodically at non-time-critical moments. Uses explicit iterator manipulation to safely erase during iteration.

### take_mytm_mutex() / release_mytm_mutex()
- **Signature:** `bool take_mytm_mutex()` / `bool release_mytm_mutex()`
- **Purpose:** Acquire/release the global timer task mutex for external (e.g., packet listener thread) synchronization.
- **Inputs:** None
- **Outputs/Return:** `true` on success, `false` on SDL error.
- **Side effects:** Locks/unlocks `sTMTaskMutex`; logs anomaly on failure.
- **Calls:** `SDL_LockMutex()`, `SDL_UnlockMutex()`, `SDL_GetError()`, `logAnomaly()`
- **Notes:** Logging in these functions is noted as potentially unsafe in multi-threaded context, but acceptable for debugging.

### myTMDumpProfile() (DEBUG only)
- **Signature:** `void myTMDumpProfile(myTMTask* inTask)`
- **Purpose:** Log profiling statistics for a timer task.
- **Inputs:** `inTask` ΓÇô pointer to task (or NULL).
- **Outputs/Return:** None
- **Side effects:** Calls `logDump()` macros.
- **Calls:** `logDump()` (via macros `DUMPIT_ZU`, `DUMPIT_ZS`)

## Control Flow Notes

**Initialization:** `mytm_initialize()` must be called once before using timer tasks.

**Task Creation:** `myXTMSetup()` spawns `thread_loop()` in a new SDL thread with boosted priority.

**Periodic Execution:** `thread_loop()` implements a tight loop:
1. Sleep for `(period - drift)` to maintain schedule
2. Detect missed deadlines if delay Γëñ 0
3. Update drift for next iteration
4. Lock mutex, call user callback, unlock
5. Loop until `mKeepRunning == false`

**Task Removal & Cleanup:** `myTMRemove()` signals task to stop (non-blocking). `myTMCleanup()` periodically reaps stopped tasks.

**Shutdown:** Not explicitly handled; mutex is never destroyed (process exit cleanup).

## External Dependencies
- **SDL2:** `SDL_Thread`, `SDL_CreateThread()`, `SDL_WaitThread()`, `SDL_mutex`, `SDL_LockMutex()`, `SDL_UnlockMutex()`, `SDL_GetError()`
- **Timing functions:** `machine_tick_count()`, `sleep_for_machine_ticks()` (defined elsewhere)
- **Priority:** `BoostThreadPriority()` from `thread_priority_sdl.h` (defined elsewhere)
- **Logging:** `logWarning()`, `logAnomaly()`, `logDump()` from `Logging.h`
- **Standard library:** `<atomic>`, `<vector>`
