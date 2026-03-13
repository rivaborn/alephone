# Source_Files/CSeries/csmisc_sdl.cpp

## File Purpose
SDL-based implementation of timing, sleep, and input-wait primitives for the Aleph One game engine. Provides tick-counting utilities with optional debug slow-motion mode (`TIME_SKEW`) and blocking input detection with timeout.

## Core Responsibilities
- Maintain a steady-clock reference epoch for reliable, monotonic millisecond-level tick counting across the game session
- Implement thread-safe sleep operations with frame-timing adjustment support via the `TIME_SKEW` constant
- Provide processor yield to reduce busy-waiting overhead in polling loops
- Poll SDL event queue for user input (mouse, keyboard, controller) with timeout and early exit on match
- Support debug "slow motion" mode via `TIME_SKEW` without changes to call sites

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `epoch` | `std::chrono::steady_clock::time_point` | static | Reference point (saved at module init) for calculating elapsed engine ticks |
| `TIME_SKEW` | `constexpr int` | static | Debug scaling factor (default 1); increases all sleep durations for testing timing-sensitive features in slow motion |

## Key Functions / Methods

### machine_tick_count
- **Signature:** `uint64_t machine_tick_count(void)`
- **Purpose:** Return elapsed milliseconds since engine start, scaled by `TIME_SKEW` for debug slow-motion support
- **Inputs:** None
- **Outputs/Return:** Uint64 millisecond count
- **Side effects:** None (pure; reads only `steady_clock`)
- **Calls:** `std::chrono::steady_clock::now()`, `std::chrono::duration_cast<>`
- **Notes:** Primary clock source for the engine; guaranteed monotonic; all engine timing should derive from this function

### sleep_for_machine_ticks
- **Signature:** `void sleep_for_machine_ticks(uint32 ticks)`
- **Purpose:** Block calling thread for a specified duration in engine ticks (milliseconds, scaled by `TIME_SKEW`)
- **Inputs:** Uint32 tick count
- **Outputs/Return:** None
- **Side effects:** Blocks thread for ~ticks ├ù TIME_SKEW milliseconds
- **Calls:** `std::this_thread::sleep_for()`
- **Notes:** Inverse semantics of `machine_tick_count()`; when TIME_SKEW=1, sleep(1000) blocks for ~1 second of engine time

### sleep_until_machine_tick_count
- **Signature:** `void sleep_until_machine_tick_count(uint64_t ticks)`
- **Purpose:** Block calling thread until an absolute tick count is reached (scaled by `TIME_SKEW`)
- **Inputs:** Uint64 target tick count
- **Outputs/Return:** None
- **Side effects:** Blocks until `steady_clock::now()` reaches `time_point(milliseconds(ticks ├ù TIME_SKEW))`
- **Calls:** `std::this_thread::sleep_until()`
- **Notes:** Interprets input as absolute steady_clock time, not relative to the game's `epoch`; typically used in frame-locking code to synchronize to a calculated future deadline

### yield
- **Signature:** `void yield(void)`
- **Purpose:** Yield processor to allow other threads to run; trivial wrapper around standard threading primitive
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Temporarily suspends calling thread; scheduler may switch to other threads
- **Calls:** `std::this_thread::yield()`

### wait_for_click_or_keypress
- **Signature:** `bool wait_for_click_or_keypress(uint32 ticks)`
- **Purpose:** Block and poll for user input (mouse click, key press, or controller button press) within a timeout window
- **Inputs:** Uint32 timeout duration in engine ticks
- **Outputs/Return:** Bool; `true` if matching input detected, `false` if timeout elapsed
- **Side effects:** Drains SDL event queue for mouse/key/controller events; blocks thread
- **Calls:** `machine_tick_count()`, `SDL_WaitEventTimeout()`
- **Notes:** Loop exits early on match; `SDL_WaitEventTimeout()` is recomputed each iteration with remaining ticks to improve timeout accuracy

## Control Flow Notes
These functions form the timing substrate for the game loop. `machine_tick_count()` is the primary clock; `sleep_for_machine_ticks()` and `sleep_until_machine_tick_count()` enforce frame pacing; `wait_for_click_or_keypress()` is used in menus or cinematics to pause for acknowledgment. The `TIME_SKEW` constant allows non-invasive debug slow-motion. No explicit init/shutdown required; `epoch` is initialized at static load time.

## External Dependencies
- **`<chrono>`** ΓÇô C++11 steady-clock utilities
- **`<thread>`** ΓÇô C++11 threading (`this_thread::sleep_for`, `sleep_until`, `yield`)
- **`"cseries.h"`** ΓÇô Game engine common types and SDL2 includes (defines `uint32`, `uint64_t`, SDL event types)
- **`"Logging.h"`** ΓÇô Logging framework (included but unused in this file)
- **SDL2 event API** (via cseries.h) ΓÇô `SDL_Event`, `SDL_WaitEventTimeout`, `SDL_MOUSEBUTTONDOWN`, `SDL_KEYDOWN`, `SDL_CONTROLLERBUTTONDOWN`
