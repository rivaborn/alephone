# Source_Files/Misc/vbl.cpp

## File Purpose

Manages vertical-blank timing, keyboard/input polling, and game recording/replay functionality. Originally handling VBL interrupts, it now acts as a periodic timer task (30 Hz) that captures player input, converts it to action flags, and either enqueues them for live play or records them to disk. Also responsible for replaying recorded games from file.

## Core Responsibilities

- Install and manage a timer task that fires 30 times per second (`input_controller`)
- Poll keyboard, mouse, and joystick state via SDL; translate to unified action flags
- Handle special input modes: double-click detection, latched keys, hotkey sequences
- Record action flags to disk in chunked, run-length-encoded format during recording
- Load and playback action flags from recording files during replay
- Manage replay speed control and pause/resume
- Serialize/deserialize recording metadata (player starts, game data, map checksums)
- Support movie export integration by spacing input updates to match target FPS

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `replay_private_data` | struct | Global state for recording/replaying: queues, file handles, playback speed, extension headers |
| `ActionQueue` | struct | Circular buffer of uint32 action flags per player |
| `special_flag_data` | struct | Metadata for latched/double-click input flags: type, persistence counter |
| `recording_header` | struct | Binary format: game metadata (num players, level, map checksum, player starts, game data) |
| `recording_extension_header` | struct | Optional extension data (saved-game WAD attachment, length) |
| `player_start_data` | struct | Player team, color, identifier, name (from map.h) |
| `game_data` | struct | Game configuration: type, options, difficulty, time/kill limits (from map.h) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `heartbeat_count` | int32 | static | Game tick counter; incremented each input frame |
| `input_task_active` | bool | static | Whether input polling is currently enabled |
| `input_task` | timer_task_proc | static | Handle to installed timer task |
| `replay` | replay_private_data | static | Global replay state: recording queue array, file spec, playback mode flags |
| `FilmFileSpec` | FileSpecifier | static | Current recording/replay file path |
| `FilmFile` | OpenedFile | static | Handle to open recording/replay file |
| `tm_func` | timer_func | static | Pointer to installed timer callback (currently only `input_controller`) |
| `tm_period` | uint64_t | static | Milliseconds between timer invocations |
| `tm_last`, `tm_accum` | uint64_t | static | Accumulator for timer scheduling |
| `hotkey_sequence[3]` | uint32_t | static | Hotkey decoding state machine (3-frame sequence) |

## Key Functions / Methods

### initialize_keyboard_controller
- **Signature:** `void initialize_keyboard_controller(void)`
- **Purpose:** Set up input system: allocate action queues, install timer task, register cleanup handler.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Allocates `replay.recording_queues` (array of ActionQueues); installs timer task; registers `atexit()` handler; calls `enter_mouse()`.
- **Calls:** `obj_clear()`, `install_timer_task()`, `atexit()`, `new[]`, `enter_mouse()`.
- **Notes:** Called once at startup. Sets `heartbeat_count = 0`, `input_task_active = false`.

### input_controller
- **Signature:** `bool input_controller(void)`
- **Purpose:** Main timer task: process input from keyboard/replay/network and enqueue action flags. Called ~30 times per second.
- **Inputs:** None (reads global state: `replay`, `dynamic_world`, `input_task_active`, `Movie::instance()`).
- **Outputs/Return:** `true` (reschedule this task).
- **Side effects:** Increments `heartbeat_count`; modifies replay phase/speed state; calls `parse_keymap()`, `process_action_flags()`, `pull_flags_from_recording()`.
- **Calls:** `parse_keymap()`, `process_action_flags()`, `pull_flags_from_recording()`, `MAX()`, `set_game_state()`.
- **Notes:** Enforces heartbeat Γëñ dynamic_world tick_count + MAXIMUM_TIME_DIFFERENCE. In replay mode, respects replay_speed (-5 to +5); speed Γëñ 0 pauses (negative speed causes phase-based stepping).

### parse_keymap
- **Signature:** `uint32 parse_keymap(void)`
- **Purpose:** Poll current SDL keyboard/mouse/joystick state and return composite action flags.
- **Inputs:** None (reads SDL input state, `input_preferences`, Console, local_player).
- **Outputs/Return:** uint32 action flags (bitfield of movement, aiming, weapon, etc.).
- **Side effects:** Updates `special_flags[]` persistence counters; advances hotkey_sequence state machine; calls mouse/joystick helper functions; applies input modifiers (run toggle, swim/sink interchange, aim inversion).
- **Calls:** `SDL_GetKeyboardState()`, `mouse_buttons_become_keypresses()`, `joystick_buttons_become_keypresses()`, `encode_hotkey_sequence()`, `process_aim_input()`, `process_joystick_axes()`, `Console::instance()`, `build_terminal_action_flags()`.
- **Notes:** Handles double-click detection and latched keys via special_flags persistence. Applies input preferences (interchange run/walk, inverted mouse, etc.). Terminal mode bypasses normal key parsing.

### process_action_flags
- **Signature:** `void process_action_flags(short player_identifier, const uint32 *action_flags, short count)`
- **Purpose:** Enqueue action flags and optionally record them.
- **Inputs:** player_identifier, pointer to action_flags array, count of flags.
- **Outputs/Return:** None.
- **Side effects:** Calls `record_action_flags()` if recording; always calls `GetRealActionQueues()->enqueueActionFlags()`.
- **Calls:** `record_action_flags()`, `GetRealActionQueues()->enqueueActionFlags()`.
- **Notes:** Gateway between input polling and game state.

### start_recording, stop_recording
- **Signature:** `void start_recording(void)`, `void stop_recording(void)`
- **Purpose:** Open/close recording file and manage replay file format.
- **Inputs/Outputs:** None.
- **Side effects:** Creates/deletes file; writes/updates recording headers; saves action flag chunks; handles optional extension headers (saved-game WAD).
- **Calls:** `get_recording_filedesc()`, `FilmFileSpec.Create()`, `FilmFileSpec.Open()`, `pack_recording_header()`, `save_recording_queue_chunk()`, `pack_recording_extension_header()`, `FilmFile.Write()`, `FilmFile.SetPosition()`.
- **Notes:** `start_recording()` requires `replay.header` to be set first via `set_recording_header_data()`. `stop_recording()` appends extension headers if `replay.saved_wad_data` is present.

### setup_for_replay_from_file
- **Signature:** `bool setup_for_replay_from_file(FileSpecifier& File, uint32 map_checksum, bool prompt_to_export)`
- **Purpose:** Open, validate, and prepare a replay file for playback.
- **Inputs:** File specifier, map checksum (unused), prompt flag.
- **Outputs/Return:** `true` if successful.
- **Side effects:** Opens file; reads header; unpacks recording metadata; allocates disk cache buffer; calls `handle_replay_extension()` if extra data present; calls `use_map_file()` or `Movie::instance()->PromptForRecording()`.
- **Calls:** `FilmFileSpec.Open()`, `FilmFile.Read()`, `unpack_recording_header()`, `handle_replay_extension()`, `use_map_file()`, `alert_user()`.
- **Notes:** Sets `replay.game_is_being_replayed = true`. Disables cheat restrictions in replays.

### save_recording_queue_chunk, read_recording_queue_chunks
- **Signature:** `void save_recording_queue_chunk(short player_index)`, `static void read_recording_queue_chunks(void)`
- **Purpose:** Serialize action queue to disk (run-length encoded) and deserialize from disk.
- **Inputs:** (save) player index; (read) none (reads global replay state).
- **Outputs/Return:** None.
- **Side effects:** (save) Reads from `replay.recording_queues[player_index]`, writes to FilmFile, updates `replay.header.length`. (read) Fills queues from FilmFile/resource data.
- **Calls:** `get_player_recording_queue()`, `ValueToStream()`, `StreamToValue()`, `FilmFile.Write()`, `FilmFile.Read()`, `vblFSRead()`.
- **Notes:** (save) Run-length encodes pairs of (count, flag). (read) Handles both in-memory (resource_data) and file-based playback; detects END_OF_RECORDING_INDICATOR.

### vblFSRead
- **Signature:** `static bool vblFSRead(OpenedFile& File, int32 *count, void *dest, bool& HitEOF)`
- **Purpose:** Buffered read from replay file with EOF detection.
- **Inputs:** File handle, desired byte count, destination buffer.
- **Outputs/Return:** `true` if read succeeded (may be partial if EOF hit); updates count and HitEOF flag.
- **Side effects:** Maintains `replay.fsread_buffer`, `replay.bytes_in_cache`, `replay.location_in_cache`.
- **Calls:** `File.GetPosition()`, `File.Read()`, `File.GetLength()`.
- **Notes:** Implements double-buffering to reduce I/O calls. Detects EOF by comparing bytes read vs. requested.

### pack_recording_header, unpack_recording_header
- **Signature:** `uint8 *pack_recording_header(uint8 *Stream, recording_header *Objects, size_t Count)`, and unpack variant.
- **Purpose:** Serialize/deserialize recording metadata to/from byte stream.
- **Inputs:** Byte stream, objects array, count.
- **Outputs/Return:** Pointer to next position in stream.
- **Side effects:** None (reads/writes streams).
- **Calls:** `ValueToStream()`, `StreamToValue()`, `PlayerStartToStream()`, `StreamToPlayerStart()`, `GameDataToStream()`, `StreamToGameData()`.
- **Notes:** Handles arrays of player_start_data and game_data. Asserts that consumed bytes match expected size.

### increment_heartbeat_count, sync_heartbeat_count
- **Signature:** `void increment_heartbeat_count(int value)`, `void sync_heartbeat_count(void)`
- **Purpose:** Advance or resync game tick counter.
- **Inputs:** (increment) tick delta; (sync) none.
- **Outputs/Return:** None.
- **Side effects:** Modifies `heartbeat_count`.
- **Notes:** `sync_heartbeat_count()` used after level loads to align with `dynamic_world->tick_count`.

### execute_timer_tasks
- **Signature:** `void execute_timer_tasks(uint64_t time)`
- **Purpose:** Called from main event loop to invoke timer tasks at appropriate intervals.
- **Inputs:** Current time (milliseconds).
- **Outputs/Return:** None.
- **Side effects:** Invokes `tm_func()` (currently `input_controller`) zero or more times depending on accumulated time.
- **Calls:** `tm_func()`, `get_fps_target()`, `mouse_idle()`.
- **Notes:** In movie-export mode, throttles to match FPS target. Otherwise uses accumulator for proper 30 Hz scheduling.

## Control Flow Notes

**Initialization:**
1. `initialize_keyboard_controller()` called at startup; installs `input_controller` as 30 Hz timer task.

**Per-frame (from main event loop):**
1. `execute_timer_tasks(current_time)` accumulates time and calls `input_controller()` when interval elapses.
2. `input_controller()` checks game state:
   - If **replaying**: calls `pull_flags_from_recording()` to load action flags from file/memory.
   - If **recording**: calls `parse_keymap()` ΓåÆ `process_action_flags()` ΓåÆ `record_action_flags()`.
   - If **live play**: calls `parse_keymap()` ΓåÆ `process_action_flags()` (no recording).
3. Periodically (from game loop), `check_recording_replaying()` triggers chunk save/load via threshold checks.

**Shutdown:**
1. `atexit()` handler calls `remove_input_controller()`, which stops recording or closes replay file.

## External Dependencies

- **SDL2** (`SDL.h`, `SDL_Scancode`, `SDL_GetKeyboardState()`, `SDL_Scancode` enums) ΓÇö Input polling.
- **FileHandler** (`FileSpecifier`, `OpenedFile`) ΓÇö File I/O abstraction.
- **ActionQueues** (`GetRealActionQueues()`, `enqueueActionFlags()`) ΓÇö Action flag queueing.
- **Player & Physics** (`local_player_index`, `local_player`, `player_start_data`, `game_data`) ΓÇö Player state (from player.h, map.h).
- **Console** (`Console::instance()->input_active()`) ΓÇö Determines if input should be captured.
- **Mouse & Joystick** (`mouse_buttons_become_keypresses()`, `joystick_buttons_become_keypresses()`, `process_aim_input()`, `process_joystick_axes()`) ΓÇö Input translation.
- **Movie** (`Movie::instance()->IsRecording()`, `PromptForRecording()`) ΓÇö Movie export integration.
- **Preferences** (`input_preferences`, `graphics_preferences`, `get_fps_target()`) ΓÇö User configuration.
- **Map/Game World** (`dynamic_world`, `use_map_file()`) ΓÇö Game state queries.
- **Logging** (`logContext()`, `logError()`, `logAnomaly()`) ΓÇö Diagnostics.
- **Packing utilities** (`ValueToStream()`, `StreamToValue()`, `BytesToStream()`, `StreamToBytes()`) ΓÇö Binary serialization.
