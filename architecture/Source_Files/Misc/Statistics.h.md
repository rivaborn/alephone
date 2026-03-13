# Source_Files/Misc/Statistics.h

## File Purpose
Defines `StatsManager`, a singleton class that collects game statistics from Lua scripts and uploads them asynchronously. Manages a queue of statistics entries and uses SDL2 threading to avoid blocking the main game loop.

## Core Responsibilities
- Implements singleton pattern for global stats manager access
- Queues and processes game statistics (options and parameters)
- Manages asynchronous stats collection via a background thread
- Synchronizes thread-safe access to the stats queue using SDL2 mutex
- Provides status queries (`Busy()`) and completion synchronization (`Finish()`)
- Integrates with the dialog system for upload status UI

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `StatsManager` | class | Singleton manager for collecting and uploading game statistics |
| `Entry` | struct (private nested) | Holds a single stats entry with key-value option and parameter maps |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `instance_` | `StatsManager*` | static (within `instance()`) | Singleton instance of StatsManager |
| `entries_` | `std::queue<Entry, std::list<Entry>>` | member | Queue of pending statistics entries to process |
| `thread_` | `SDL_Thread*` | member | Handle to background worker thread |
| `entry_mutex_` | `SDL_mutex*` | member | Synchronization primitive for thread-safe queue access |
| `run_` | `bool` | member | Control flag for background thread lifecycle |
| `busy_` | `bool` | member | Status flag indicating active processing/upload |

## Key Functions / Methods

### instance() [static]
- **Signature:** `static StatsManager* instance()`
- **Purpose:** Provides lazy-initialized singleton access to the stats manager
- **Inputs:** None
- **Outputs/Return:** Pointer to the unique `StatsManager` instance
- **Side effects:** Allocates manager on first call (no deallocation visible)
- **Notes:** Classic static-local singleton pattern; thread-safety of initialization not explicit

### Process()
- **Signature:** `void Process()`
- **Purpose:** Grabs current stats from Lua script and enqueues them for upload
- **Inputs:** None (accesses Lua state implicitly)
- **Outputs/Return:** None
- **Side effects:** Modifies `entries_` queue; likely locks `entry_mutex_`
- **Calls:** Not shown in header; likely calls into Lua bindings

### Busy()
- **Signature:** `bool Busy()`
- **Purpose:** Queries whether stats are currently being written or uploaded
- **Inputs:** None
- **Outputs/Return:** `busy_` flag state
- **Notes:** Non-blocking query; may be called from main thread to avoid blocking on upload

### Finish()
- **Signature:** `void Finish()`
- **Purpose:** Likely waits for all pending stats to finish uploading before shutdown
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Blocks until `busy_` is false and queue is empty (inferred)
- **Notes:** Likely signals `run_` = false and joins `thread_`

### Run() [static, private]
- **Signature:** `static int Run(void*)`
- **Purpose:** Thread entry point; processes queue and uploads stats in background
- **Inputs:** `void*` context (likely `this`)
- **Outputs/Return:** Thread exit code (int)
- **Side effects:** Consumes `entries_`, communicates with upload service, manages `busy_` flag
- **Notes:** Runs while `run_` is true; likely loops on `entries_` with mutex protection

### CheckForDone() [private]
- **Signature:** `void CheckForDone(dialog* d)`
- **Purpose:** Callback to check upload completion status and update dialog UI
- **Inputs:** Dialog pointer (for UI updates)
- **Side effects:** May modify dialog state
- **Notes:** Likely used as a processing function on a modal dialog during upload

## Control Flow Notes
Typical flow:
1. **Game tick/frame**: `Process()` is called to collect current Lua stats into the queue
2. **Background thread**: `Run()` periodically wakes, locks mutex, dequeues entries, and uploads them
3. **Polling**: Main thread calls `Busy()` to check completion status
4. **Shutdown**: `Finish()` is called to block until all stats are uploaded and thread exits

The async design allows stats collection without frame-rate impact.

## External Dependencies
- **SDL2**: `SDL_mutex`, `SDL_thread` for multi-threading and synchronization
- **STL**: `std::queue`, `std::list`, `std::map`, `std::string` for data structures
- **CSeries**: `cseries.h` (project core utilities, includes)
- **Dialogs**: `sdl_dialogs.h` (dialog class for UI feedback)
- **Lua**: Referenced in comments ("grabs stats from Lua script") but bindings not visible in this header
