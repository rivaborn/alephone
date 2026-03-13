# Source_Files/Misc/vbl_definitions.h

## File Purpose
Defines core data structures and interfaces for the Aleph One game engine's recording and replay system. Provides action queue management, replay state storage, and timer task support for frame-by-frame input capture and playback.

## Core Responsibilities
- Define action queue structure for queuing player inputs during gameplay
- Define recording header and extension formats for saved game metadata
- Manage replay/recording state via the global `replay` struct
- Provide interfaces for installing/removing periodic timer tasks
- Expose helper macros for accessing player recording queues

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ActionQueue` | struct | 8-byte structure holding read/write indices and buffer pointer for player input queue |
| `recording_header` | struct | Metadata for a recording session (length, player count, level, checksums, player starts, game state) |
| `recording_extension_type` | enum class | Extension type selector; currently supports `none` or `saved_game_wad` |
| `recording_extension_header` | struct | Header for optional recording extensions (type + data length) |
| `replay_private_data` | struct | Main state container: validity flag, recording header, replay speed, playback/recording flags, action queues, file buffers, resource data, and WAD chunks |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `replay` | `struct replay_private_data` | extern (global) | Central state for active replay or recording session |
| `MAXIMUM_QUEUE_SIZE` | macro (int) | file | Maximum capacity of action queue buffer (512) |
| `SIZEOF_recording_header` | const int | file | Precomputed size of `recording_header` struct (352 bytes) |
| `SIZEOF_recording_extension_header` | const int | file | Precomputed size of `recording_extension_header` struct (8 bytes) |

## Key Functions / Methods

### get_player_recording_queue
- **Signature:** `#define get_player_recording_queue(x)  (replay.recording_queues+(x))` (release) or function prototype (debug)
- **Purpose:** Retrieve the action queue for a specific player by index.
- **Inputs:** `x` ΓÇô player index
- **Outputs/Return:** Pointer to `ActionQueue` for that player
- **Side effects:** None (macro expands to pointer arithmetic; debug version may perform bounds checking)
- **Notes:** In debug builds, a full function definition provides validation; in release, it's a simple macro for performance.

### install_timer_task
- **Signature:** `timer_task_proc install_timer_task(short tasks_per_second, bool (*func)(void))`
- **Purpose:** Install a periodic callback to execute at specified frequency.
- **Inputs:** `tasks_per_second` ΓÇô frequency; `func` ΓÇô callback function returning bool
- **Outputs/Return:** `timer_task_proc` opaque handle for later removal
- **Side effects:** Registers timer interrupt or equivalent mechanism; callback will be invoked periodically
- **Calls:** Not visible in this file (implementation in `vbl.c` or `vbl_macintosh.c`)
- **Notes:** Typically used to drive action queue updates at fixed frame rate.

### remove_timer_task
- **Signature:** `void remove_timer_task(timer_task_proc proc)`
- **Purpose:** Uninstall a previously registered timer task.
- **Inputs:** `proc` ΓÇô handle returned by `install_timer_task`
- **Outputs/Return:** None
- **Side effects:** Deregisters callback; stops periodic invocation
- **Calls:** Not visible in this file

## Control Flow Notes
This file is infrastructure for the recording/replay pipeline. During recording, `install_timer_task` likely drives periodic calls to capture input into action queues. During playback, queues are read from the `replay` state. The `recording_header` and optional `recording_extension_header` structure the save format, while `replay_private_data.fsread_buffer` and `resource_data` cache file/WAD contents in memory.

## External Dependencies
- **`#include "player.h"`** ΓÇô Defines `struct player_start_data`, `struct game_data`, and player configuration constants (`MAXIMUM_NUMBER_OF_PLAYERS`); used in `recording_header`
- **File-local use only:** Per header comment, this is consumed by `vbl.c` and `vbl_macintosh.c`; no broader interdependency inferred
