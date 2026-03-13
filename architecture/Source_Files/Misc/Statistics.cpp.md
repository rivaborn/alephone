# Source_Files/Misc/Statistics.cpp

## File Purpose
Implements `StatsManager`, which collects gameplay statistics from Lua scripts, queues them asynchronously, and uploads them to a remote statistics server via HTTP POST requests. Designed to be non-blocking by offloading uploads to a dedicated thread.

## Core Responsibilities
- Collect game statistics from Lua scripts and queue them for upload
- Manage a background upload thread using SDL threading primitives
- Generate checksums for statistics entries before transmission
- Make HTTP POST requests to the statistics server with authentication
- Provide a blocking finish dialog that waits for all pending uploads
- Synchronize queue access between main thread and upload thread using mutex

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Entry` | struct (in header) | Encapsulates a single stats record with `options` and `parameters` maps |
| `ScopedMutex` | class | RAII wrapper for SDL mutex locking/unlocking |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `entry_mutex_` | SDL_mutex\* | StatsManager member | Protects concurrent access to `entries_` queue |
| `entries_` | std::queue<Entry> | StatsManager member | Thread-safe queue of statistics to upload |
| `thread_` | SDL_Thread\* | StatsManager member | Handle to background upload thread |
| `run_` | bool | StatsManager member | Signal flag: thread runs while true, exits when false |
| `busy_` | bool | StatsManager member | Flag: true if entries queued or being uploaded, false when idle |

## Key Functions / Methods

### StatsManager::StatsManager() (constructor)
- **Signature:** `StatsManager::StatsManager() : thread_(0), run_(true), busy_(false)`
- **Purpose:** Initialize the stats manager and spawn the background upload thread.
- **Side effects (global state, I/O, alloc):** Allocates SDL mutex via `SDL_CreateMutex()`, spawns new SDL thread via `SDL_CreateThread()`.
- **Calls:** `SDL_CreateMutex()`, `SDL_CreateThread()`
- **Notes:** Singleton instance created on first call to `StatsManager::instance()`.

### StatsManager::Process()
- **Signature:** `void StatsManager::Process()`
- **Purpose:** Collect current game statistics from Lua and enqueue them for upload.
- **Inputs:** None (reads Lua globals via `CollectLuaStats()`).
- **Outputs/Return:** None.
- **Side effects:** Acquires mutex, pushes Entry onto queue, sets `busy_ = true` if successful.
- **Calls:** `CollectLuaStats()` (defined elsewhere), `ScopedMutex` constructor/destructor, `std::queue::push()`.
- **Notes:** Called each frame if stats collection is enabled; non-blocking.

### StatsManager::Finish()
- **Signature:** `void StatsManager::Finish()`
- **Purpose:** Block the caller until all queued statistics are uploaded (if busy), then signal thread to shut down.
- **Side effects:** If `busy_`, spawns a modal dialog with progress indication and "CANCEL" button. Sets `run_ = false` and waits for thread via `SDL_WaitThread()`.
- **Calls:** Dialog construction (w_static_text, w_spacer, w_button, vertical_placer), `d.set_processing_function()` with lambda binding, `SDL_WaitThread()`.
- **Notes:** Modal UI blocks until upload completes or user cancels; canvas for graceful shutdown.

### StatsManager::CheckForDone(dialog\* d)
- **Signature:** `void StatsManager::CheckForDone(dialog* d)`
- **Purpose:** Dialog processing callback; checks if upload is complete and closes the dialog if so.
- **Inputs:** Pointer to the progress dialog.
- **Outputs/Return:** None.
- **Side effects:** If `!busy_`, sets `run_ = false`, waits for thread, closes dialog.
- **Calls:** `SDL_WaitThread()`, `dialog::quit()`.
- **Notes:** Invoked repeatedly while dialog is active; blocks shutdown until queue drains.

### StatsManager::Run(void \*pv) (static)
- **Signature:** `static int StatsManager::Run(void *pv)`
- **Purpose:** Main loop for the background upload thread; dequeues statistics and sends them via HTTP.
- **Inputs:** `pv` = opaque pointer to StatsManager instance (cast internally).
- **Outputs/Return:** Always returns 0 (SDL thread entry point convention).
- **Side effects:** 
  - Dequeues entries in a loop while `run_` is true.
  - Adds platform, username, password, and session ID to each entry.
  - Computes simple checksum over all keys and values.
  - Makes HTTP POST request to `A1_STATSERVER_ADD_URL`.
  - Sets/clears `busy_` flag based on queue state.
  - Sleeps 0.2s if queue is empty (throttles busy-wait).
- **Calls:** `ScopedMutex`, `std::make_unique<Entry>()`, `std::queue::pop()`, `checksum_string()`, `std::ostringstream`, `HTTPClient::Post()`, `sleep_for_machine_ticks()`.
- **Notes:** 
  - Runs until `run_` is signaled false.
  - Directly accesses `dynamic_world->player_count`, `network_preferences` globals.
  - Credentials sent in plaintext; checksum is non-cryptographic (simple byte sum).

### checksum_string(const std::string& s) (static)
- **Signature:** `static uint32 checksum_string(const std::string& s)`
- **Purpose:** Compute a simple checksum by summing byte values.
- **Inputs:** Reference to string.
- **Outputs/Return:** 32-bit unsigned checksum (sum of all bytes cast to unsigned char).
- **Notes:** Not cryptographically secure; used only for data integrity check and simple authentication.

### ScopedMutex (class)
- **Constructor:** `ScopedMutex(SDL_mutex* mutex) : mutex_(mutex) { SDL_LockMutex(mutex_); }`
- **Destructor:** `~ScopedMutex() { SDL_UnlockMutex(mutex_); }`
- **Purpose:** RAII wrapper ensuring mutex is locked on construction and unlocked on destruction.
- **Notes:** Exception-safe; no dynamic allocation.

## Control Flow Notes
- **Frame/Process phase:** Main thread calls `Process()` to collect stats and enqueue them.
- **Shutdown phase:** Main thread calls `Finish()`, which blocks until the upload thread completes all entries and sets `run_ = false`.
- **Background thread:** `Run()` continuously dequeues entries and uploads them asynchronously; sleeps briefly if queue is empty to avoid busy-waiting.
- **Synchronization:** Mutex protects the queue; `busy_` flag signals completion to the Finish dialog.

## External Dependencies
- **SDL2:** `SDL_CreateMutex()`, `SDL_LockMutex()`, `SDL_UnlockMutex()`, `SDL_CreateThread()`, `SDL_WaitThread()`
- **Custom game engine:** 
  - `CollectLuaStats()` ΓÇö populates options/parameters from Lua
  - `HTTPClient::Post()` ΓÇö sends POST request
  - `dynamic_world->player_count`, `network_preferences` ΓÇö global game state
  - `NetSessionIdentifier()` ΓÇö returns multiplayer session ID
  - `dialog`, `w_static_text`, `w_spacer`, `w_button`, `vertical_placer` ΓÇö UI dialogs
  - `A1_STATSERVER_ADD_URL`, `A1_DISPLAY_PLATFORM` ΓÇö version/config macros
- **Standard library:** `std::queue`, `std::list`, `std::map`, `std::string`, `std::unique_ptr`, `std::ostringstream`, `std::bind`
