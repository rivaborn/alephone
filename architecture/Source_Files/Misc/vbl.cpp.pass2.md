# Source_Files/Misc/vbl.cpp - Enhanced Analysis

## Architectural Role

vbl.cpp is the **temporal synchronization and input orchestration hub** of Aleph One. It mediates between SDL input devices and the GameWorld simulation through a 30 Hz timer task, while simultaneously managing the recording/replay subsystem that enables demos, multiplayer determinism, and playback validation. The heartbeat counter it maintains is the canonical tick counter for the engineΓÇöall world simulation and action processing flows through this file's per-frame cadence.

## Key Cross-References

### Incoming (who depends on this file)

**Core Loop:**
- `shell.cpp` / main event loop calls `execute_timer_tasks()` every frame to trigger input polling

**Game Lifecycle:**
- `interface.cpp` calls `initialize_keyboard_controller()` at startup
- `atexit()` handler (registered here) calls `remove_input_controller()` on shutdown
- Multiple subsystems query `get_heartbeat_count()` and `get_keyboard_controller_status()` for synchronization

**Recording/Replay Integration:**
- `preferences.cpp` / `shell.cpp` call `start_recording()`, `stop_recording()`, `setup_for_replay_from_file()`
- `Movie.cpp` (video export) checks `Movie::instance()->IsRecording()` via this module's input_controller
- Game state machine calls `set_game_state(_switch_demo)` when replay ends

**Synchronization Points:**
- `marathon2.cpp` calls `sync_heartbeat_count()` after level loads to align input tick with world tick
- Physics/collision code reads `dynamic_world->tick_count` to validate heartbeat isn't drifting

### Outgoing (what this file depends on)

**Input Devices (Input Subsystem):**
- Directly polls SDL via `SDL_GetKeyboardState()`, `SDL_PumpEvents()`, `SDL_FlushEvents()`
- Calls `mouse_buttons_become_keypresses()`, `joystick_buttons_become_keypresses()` from `mouse.cpp` / `joystick.cpp`
- Reads `input_preferences` (sensitivity, deadzone, key bindings)

**Action Queue (Game Action Dispatch):**
- `GetRealActionQueues()->enqueueActionFlags()` dispatches all input to GameWorld simulation

**World Timing:**
- Reads `dynamic_world->tick_count` to enforce max time difference between heartbeat and world
- Reads `local_player_index`, `local_player` state for context

**File I/O & Recording:**
- `FileHandler.cpp` / `FileSpecifier` for file paths and directory operations
- `Packing.h` utilities (`ValueToStream`, `StreamToValue`) for binary serialization
- Uses object-oriented `OpenedFile` abstraction for buffered I/O

**Configuration & State:**
- `preferences.cpp` for input device selection, sensitivity curves, key binding tables
- `Console.cpp` to detect terminal input mode (bypasses game input)
- `graphics_preferences` to query FPS target for movie export frame spacing

## Design Patterns & Rationale

### Timer Task Pattern
- **Why:** Decouples input polling from main render loop to ensure consistent 30 Hz action sampling regardless of frame rate (render can be 60+, input stays locked at 30)
- **How:** `install_timer_task(TICKS_PER_SECOND, input_controller)` schedules callback; `execute_timer_tasks()` invoked from main loop uses accumulator to call task when interval elapses
- **Tradeoff:** Adds complexity (separate callback context) but enables decoupled timing and deterministic replay

### Circular Queue Recording
- **Why:** Recording long gameplay sessions without memory bloat; run-length encoding compresses repeated input
- **How:** `ActionQueue` structure holds `read_index`, `write_index`, and fixed `MAXIMUM_QUEUE_SIZE` buffer; `INCREMENT_QUEUE_COUNTER` macro wraps indices
- **Tradeoff:** Bounded queue can overflow (triggers dprintf warning); RLE encoding loses lossless compressed size tracking

### Replay Phase-Based Stepping
- **Why:** Allow negative replay speeds (slow-motion frame stepping) without breaking world simulation tick rate
- **How:** Static `phase` counter in `input_controller` counts down when replay_speed < 0; only pulls input when phase reaches zero; resets phase to `-(replay_speed) + 1`
- **Rationale:** Maintains 30 FPS world ticks even when replaying at 1/5 speed; frame-by-frame pause (speed = -5) keeps world paused until user advances

### Double-Buffered Disk Read Cache
- **Why:** Replay from large files would cause frame stutters if reading small chunks from disk
- **How:** `vblFSRead()` maintains `fsread_buffer`, `bytes_in_cache`, `location_in_cache`; reads `DISK_CACHE_SIZE` bytes ahead
- **Cost:** ~600 bytes per cache buffer; prevents I/O stalls during playback

## Data Flow Through This File

```
RECORDING PATH:
  SDL Events ΓåÆ SDL_GetKeyboardState() + joystick/mouse polling
      Γåô
  parse_keymap() [hotkey decode, latched keys, double-click, special flags]
      Γåô
  process_action_flags() [may call record_action_flags]
      Γåô
  record_action_flags() [write to circular queue buffer]
      Γåô
  [periodically] save_recording_queue_chunk() [RLE encode + write to disk]

REPLAY PATH:
  Disk file ΓåÆ read_recording_queue_chunks() [RLE decode into circular queue]
      Γåô
  input_controller() [respects replay_speed phase stepping]
      Γåô
  pull_flags_from_recording() [read from circular queue]
      Γåô
  GetRealActionQueues()->enqueueActionFlags()
      Γåô
  GameWorld simulation (deterministic, validated against original)

LIVE PATH:
  SDL Events ΓåÆ parse_keymap() ΓåÆ process_action_flags() ΓåÆ GetRealActionQueues()
```

**Timing Synchronization:**
- `heartbeat_count` incremented each frame; compared to `dynamic_world->tick_count`
- First frame after load allows up to `MAXIMUM_TIME_DIFFERENCE` ticks of drift; afterward requires near-lock
- Network games enforce same constraint via heartbeat messages

## Learning Notes

### Idiomatic to Aleph One / Early-2000s Game Engine Design

1. **Fixed-rate input polling:** 30 Hz input/simulation tick separated from variable-rate renderingΓÇöcommon in pre-GPU era when networking required deterministic ticks. Modern engines often use interpolation instead.

2. **Circular queue buffers:** Pre-STL era; fixed-size memory pools avoid dynamic allocation during gameplay (predictable memory behavior).

3. **Run-length encoding for compression:** Chosen because input flags are highly repetitive (same action held for many ticks); simple to decode and didn't require third-party compression library.

4. **Binary serialization with hand-written packing:** No protobuf/flatbuffers; uses stream helper functions (`ValueToStream`) and struct-to-bytes conversion. Requires manual versioning for compatibility (seen in recording_extension_header system).

5. **Global state via `static`:** Pre-C++11; no singleton pattern or dependency injection. Globals like `heartbeat_count`, `replay`, `FilmFile` are module-private but implicit.

6. **Callback-based async I/O:** File operations are synchronous, scheduled via timer callbacks rather than threads.

### Modern Alternatives

- Input: Event-driven input from OS with event queue (no polling)
- Replay: Event sourcing pattern with deterministic event replay
- Recording: Protocol buffers or Cap'n Proto instead of hand-written binary format
- Timing: Fixed timestep simulation with interpolated rendering (decouples display FPS from simulation)
- State: Snapshot-based save/load with delta compression

## Potential Issues

### 1. **Heartbeat Drift on Level Load**
- After `use_map_file()`, heartbeat and world ticks are out of sync; relies on `sync_heartbeat_count()` being called
- If sync call is missed, subsequent replays will desynced after level transition
- **Mitigation:** Assert in debug builds; code comments; sync_heartbeat_count called in known places

### 2. **Recording Queue Overflow (Silent)**
- If player input rate exceeds disk I/O rate, circular queue wraps and overwrites old input
- `vwarn()` logs mismatch but doesn't prevent data loss (game continues)
- **Impact:** Replays of long sessions may have gaps without obvious player notification
- **Modern approach:** Bounded queue would block input or drop frames explicitly

### 3. **Replay Speed Phase Calculation Fragility**
- Phase reset: `phase = -(replay_speed) + 1` is non-obvious; magic number +1
- Negative replay_speed -1 ΓåÆ phase = 2 (update every 2 frames); -5 ΓåÆ phase = 6
- If phase calculation is wrong, fast-forward/rewind is subtle jittery
- **Defense:** Unit tests for phase stepping behavior (though none visible in codebase)

### 4. **Extension Header File Position Race**
- `stop_recording()` seeks to write extension header at end of file after main data
- If another thread is reading replay file simultaneously, position inconsistency possible
- **Mitigation:** Assumes single-threaded record/replay (enforced in practice via game state machine)

### 5. **Input Preferences Applied at Poll Time**
- `parse_keymap()` reads global `input_preferences` each frame
- Changing key bindings mid-game updates immediately (could break active recording/replay consistency)
- **Design:** Recording captures raw SDL keys, then replays through current key binding table
- **Risk:** If key binding changes between record and replay, input may differ (feature or bug?)

### 6. **No Versioning for Action Flags**
- `uint32` action flags are hard-coded; adding new input flags requires binary compatibility hack
- Recording format has version field but replay doesn't validate flag version match
- **Example:** Marathon Infinity added more flags than original Marathon 2

---

## Summary

vbl.cpp is a **thin but critical bridge**: it translates continuous raw input into discrete 30 FPS action flags, enforces temporal synchronization between input and world simulation, and provides recording/replay infrastructure essential to determinism and demos. Its design reflects early-2000s constraints (no STL, fixed-size buffers, hand-rolled serialization) and would benefit from modernization (event-driven input, proper versioning) but remains fundamentally sound for the engine's single-threaded, fixed-timestep architecture.
