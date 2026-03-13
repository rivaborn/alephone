# Source_Files/RenderOther/computer_interface.h

## File Purpose
Defines the interface for rendering and managing on-screen computer/terminal interfaces that players interact with in-game. Handles terminal mode initialization, state management, user input, and serialization of terminal data (both map-resident and per-player state).

## Core Responsibilities
- Initialize and manage terminal rendering state per player
- Render terminal UI and text content with formatting (bold, italic, underline)
- Process player input during terminal interaction
- Pack/unpack terminal data for map loading and player state persistence
- Support terminal content sequences (logon, information, checkpoints, sounds, movies, tracks, teleports)
- Query and manage terminal mode activation/deactivation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `static_preprocessed_terminal_data` | struct | Binary metadata header for compiled terminal content (length, flags, line count, group/font-change counts) |
| `view_terminal_data` | struct | Viewport geometry for terminal rendering (top, left, bottom, right, vertical offset) |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_terminal_manager
- Signature: `void initialize_terminal_manager(void)`
- Purpose: One-time engine initialization for terminal subsystem
- Inputs: None
- Outputs/Return: None
- Side effects: Sets up global terminal state
- Calls: (not visible)
- Notes: Called at engine startup

### initialize_player_terminal_info
- Signature: `void initialize_player_terminal_info(short player_index)`
- Purpose: Per-player terminal state initialization
- Inputs: Player index
- Outputs/Return: None
- Side effects: Allocates/resets player-specific terminal data
- Calls: (not visible)

### enter_computer_interface
- Signature: `void enter_computer_interface(short player_index, short text_number, short completion_flag)`
- Purpose: Transition player into terminal mode with specified text content
- Inputs: Player index, text resource ID, completion flag
- Outputs/Return: None
- Side effects: Enters terminal interaction mode; suppresses normal gameplay input
- Calls: (not visible)

### _render_computer_interface
- Signature: `void _render_computer_interface(void)`
- Purpose: Frame-rate rendering of active terminal interface
- Inputs: None (reads global terminal state)
- Outputs/Return: None
- Side effects: Writes to framebuffer
- Calls: (not visible)
- Notes: Called once per frame if player in terminal mode

### update_player_for_terminal_mode
- Signature: `void update_player_for_terminal_mode(short player_index)`
- Purpose: Per-frame state update for player in terminal mode (animations, timers)
- Inputs: Player index
- Outputs/Return: None
- Side effects: Updates internal state, may trigger media playback
- Calls: (not visible)

### update_player_keys_for_terminal
- Signature: `void update_player_keys_for_terminal(short player_index, uint32 action_flags)`
- Purpose: Process player input while in terminal mode
- Inputs: Player index, action flags from input system
- Outputs/Return: None
- Side effects: Navigates terminal content, may exit terminal mode
- Calls: (not visible)

### build_terminal_action_flags
- Signature: `uint32 build_terminal_action_flags(char *keymap)`
- Purpose: Convert keymap data to action flags for terminal input
- Inputs: Keymap buffer
- Outputs/Return: Packed action flags
- Calls: (not visible)

### player_in_terminal_mode
- Signature: `bool player_in_terminal_mode(short player_index)`
- Purpose: Query whether player is currently in terminal interaction mode
- Inputs: Player index
- Outputs/Return: Boolean
- Calls: (not visible)

### unpack_map_terminal_data / pack_map_terminal_data
- Signature: `extern void unpack_map_terminal_data(uint8 *Stream, size_t Count)` / `pack_map_terminal_data(uint8 *Stream, size_t Count)`
- Purpose: Deserialize/serialize terminal content metadata from map file resource stream
- Inputs: Byte stream, byte count
- Outputs/Return: None (modifies global terminal data cache)
- Side effects: Populates compiled terminal cache for all terminals in current map
- Notes: "Map terminal" = data read from map file

### unpack_player_terminal_data / pack_player_terminal_data
- Signature: `uint8 *unpack_player_terminal_data(uint8 *Stream, size_t Count)` / `pack_player_terminal_data(uint8 *Stream, size_t Count)`
- Purpose: Serialize/deserialize per-player terminal state for save/load
- Inputs: Byte stream, byte count
- Outputs/Return: Pointer to next unread/unwritten byte (allows chaining)
- Notes: "Player terminal" = player's current terminal viewing position/state

### calculate_packed_terminal_data_length
- Signature: `extern size_t calculate_packed_terminal_data_length(void)`
- Purpose: Calculate byte size of packed terminal state (for save file pre-allocation)
- Outputs/Return: Byte count
- Calls: (not visible)

### clear_compiled_terminal_cache
- Signature: `void clear_compiled_terminal_cache()`
- Purpose: Flush compiled terminal content cache (e.g., on level unload)
- Outputs/Return: None
- Side effects: Frees terminal resources
- Calls: (not visible)

## Control Flow Notes
**Initialization:** `initialize_terminal_manager()` (engine startup) ΓåÆ `initialize_player_terminal_info()` per player (level start)

**Per Frame:** `update_player_for_terminal_mode()` (state/animation), then `_render_computer_interface()` (if in terminal mode)

**Input:** `update_player_keys_for_terminal()` drives navigation; may call `abort_terminal_mode()` on exit

**Serialization:** `unpack_map_terminal_data()` loads map terminals; `unpack/pack_player_terminal_data()` handle save/load

The file documents a text format with directives (#LOGON, #SUCCESS, #FAILURE, #INFORMATION, #CHECKPOINT, #SOUND, #MOVIE, #TRACK, #INTERLEVEL TELEPORT, #INTRALEVEL TELEPORT, #PICT) and embedded formatting codes ($B/$b for bold, $I/$i for italic, $U/$u for underline), which are preprocessed into the binary `static_preprocessed_terminal_data` format.

## External Dependencies
- **cstypes.h**: Fixed-size integer types (`int16`, `uint32`, `uint8`, etc.)
