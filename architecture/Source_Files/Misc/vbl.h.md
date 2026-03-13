# Source_Files/Misc/vbl.h

## File Purpose
Header for the replay recording and playback subsystem in the game engine. Manages recording game state, input, and header metadata to file, and setting up replay playback from saved replay files. Synchronizes with vertical-blank interrupts for frame-accurate input capture.

## Core Responsibilities
- Replay file setup and initialization (from file or random resource)
- Recording state management and header data (players, level, map checksum, game settings)
- Recording input capture and WAD data buffering
- Keyboard controller initialization and keymap parsing
- Recording file lifecycle (find, move, get file descriptor)
- Debug support for recording replay flag streams

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `recorded_flag` | struct | (Debug only) Tracks a recorded action flag and associated player index |

## Global / File-Static State
None.

## Key Functions / Methods

### setup_for_replay_from_file
- Signature: `bool setup_for_replay_from_file(FileSpecifier& File, uint32 map_checksum, bool prompt_to_export = false)`
- Purpose: Initialize replay playback from a specified file
- Inputs: File specifier, expected map checksum, optional export prompt flag
- Outputs/Return: True if setup succeeds
- Side effects: Loads replay file, validates map checksum
- Notes: Optional export dialog may be shown to user

### start_recording
- Signature: `void start_recording(void)`
- Purpose: Begin recording game input and state
- Side effects: Initializes recording buffers and state

### set_recording_header_data / get_recording_header_data
- Purpose: Store/retrieve replay metadata (player count, level, map checksum, version, player starts, game state)
- Inputs (set): Header fields; Outputs (get): Pointers to metadata fields
- Side effects: Copies data to/from recording buffer

### input_controller
- Signature: `bool input_controller(void)`
- Purpose: Process input and record/playback frame input
- Outputs/Return: True if input processed successfully
- Side effects: Updates recording state or playback state on each frame

### increment_heartbeat_count
- Signature: `void increment_heartbeat_count(int value = 1)`
- Purpose: Increment frame/timing counter (tied to VBL)
- Inputs: Delta to add (default 1)
- Notes: Synchronizes with vertical-blank timing

### initialize_keyboard_controller
- Signature: `void initialize_keyboard_controller(void)`
- Purpose: Initialize keyboard input system

### parse_keymap
- Signature: `uint32 parse_keymap(void)`
- Purpose: Parse keyboard mapping configuration
- Outputs/Return: Parsed keymap value

### Helper functions (summary)
- `find_replay_to_use()` ΓÇô Locate a replay file (prompt user if requested)
- `get_recording_filedesc()` ΓÇô Retrieve file descriptor for active recording
- `move_replay()` ΓÇô Move/rename replay file
- `setup_replay_from_random_resource()` ΓÇô Load replay from bundled resource
- `set_recording_saved_wad_data()` ΓÇô Buffer WAD snapshot during recording

### Debug functions (DEBUG_REPLAY only)
- `open_stream_file()` / `close_stream_file()` ΓÇô Manage debug trace file
- `write_flags()` ΓÇô Write recorded flag stream to debug file
- `debug_stream_of_flags()` ΓÇô Log action flag for debugging

## Control Flow Notes
Operates as a **frame-synchronized input/timing subsystem**, likely called once per game frame (synced to VBL). In recording mode, captures input and game state; in playback mode, restores input from the replay file and advances heartbeat. Maps and validates against `map_checksum` to ensure replay consistency.

## External Dependencies
- `FileHandler.h` ΓÇô `FileSpecifier` class for file I/O abstraction
- Implicit: `struct player_start_data`, `struct game_data` (defined elsewhere in engine)
- `std::vector<byte>` ΓÇô for WAD data buffering
