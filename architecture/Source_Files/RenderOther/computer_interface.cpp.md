# Source_Files/RenderOther/computer_interface.cpp

## File Purpose
Implements the in-game computer terminal interface system where players read messages, view maps, and trigger events. Manages terminal content compilation, rendering, text layout, user input, and state persistence across save/load cycles.

## Core Responsibilities
- Load, parse, and compile Marathon-format terminal text with embedded formatting codes
- Render terminal UI with text, images (PICT resources), checkpoint maps, and visual effects
- Handle player input (keyboard/joystick) for terminal navigation (scroll, page, abort)
- Manage per-player terminal state (current group, line, phase, completion status)
- Pack/unpack terminal data to/from save files and network streams
- Calculate text layout for multiple fonts with word-wrap and line breaking
- Dispatch to specialized rendering for different terminal content types (logon, information, checkpoint, teleport, etc.)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `terminal_text_t` | struct | Single terminal's compiled content: groupings, font changes, text bytes |
| `player_terminal_data` | struct | Per-player state: current group/line, phase, completion flag |
| `terminal_groupings` | struct | One group in a terminal: type, permutation, text span, max line count |
| `text_face_data` | struct | Font styling at a text offset: bold/italic/underline flags, color index |
| `MarathonTerminalCompiler` | class | Parses Marathon text format and compiles to internal representation |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `map_terminal_text` | `vector<terminal_text_t>` | static | All terminals loaded from map |
| `player_terminals` | `player_terminal_data*` | static | Per-player terminal state (heap-allocated array) |
| `current_pixel` | `uint32` | static | Current text rendering color as SDL pixel value |
| `current_style` | `uint16` | static | Current text style (bold/italic/underline) |
| `terminal_keys` | `terminal_key[]` | static | Keycode-to-action mappings for terminal input |

## Key Functions / Methods

### initialize_terminal_manager
- **Signature:** `void initialize_terminal_manager(void)`
- **Purpose:** Set up per-player terminal state array at game start.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Allocates `player_terminals` array, initializes all players to `_no_terminal_state`.
- **Calls:** `new`, `objlist_clear()`
- **Notes:** Called once during engine initialization.

### enter_computer_interface
- **Signature:** `void enter_computer_interface(short player_index, short text_number, short completion_flag)`
- **Purpose:** Start terminal interaction for a player; show logon screen and select first group.
- **Inputs:** Player index, terminal ID, level-completion state.
- **Outputs/Return:** None.
- **Side effects:** Sets player's terminal state to `_reading_terminal`, adjusts `lines_per_page` if needed, plays logon sound if no terminal data.
- **Calls:** `get_indexed_terminal_data()`, `calculate_lines_per_page()`, `play_object_sound()`, `next_terminal_group()`
- **Notes:** Returns early if terminal data not found; graceful degradation for missing content.

### _render_computer_interface
- **Signature:** `void _render_computer_interface(void)`
- **Purpose:** Render the current terminal to screen if dirty; main rendering dispatch for all terminal types.
- **Inputs:** None (uses global `current_player_index`).
- **Outputs/Return:** None.
- **Side effects:** Clears dirty flag, clipping rectangle manipulation; calls group-specific renderers.
- **Calls:** `get_player_terminal_data()`, `get_indexed_terminal_data()`, `get_indexed_grouping()`, `draw_logon_text()`, `draw_computer_text()`, `present_checkpoint_text()`, `display_picture_with_text()`, `fill_terminal_with_static()`, `draw_terminal_borders()`
- **Notes:** Dispatches based on `current_group->type` enum; returns early if not dirty or state is `_no_terminal_state`.

### _draw_computer_text
- **Signature:** `static void _draw_computer_text(char *base_text, short group_index, Rect *bounds, terminal_text_t *terminal_text, short current_line)`
- **Purpose:** Core text rendering loop; handles line breaking, font changes, and drawing text within bounds.
- **Inputs:** Text buffer, group index, drawing bounds, terminal data, starting line number.
- **Outputs/Return:** None.
- **Side effects:** Modifies `current_style`, renders text to `draw_surface`.
- **Calls:** `get_indexed_grouping()`, `calculate_line()`, `get_indexed_font_changes()`, `set_text_face()`, `draw_line()`
- **Notes:** Handles font-change lookups for mid-text styling; skips lines before `current_line` to support scrolling.

### calculate_line
- **Signature:** `static bool calculate_line(char *base_text, short width, short start_index, short text_end_index, short *end_index)`
- **Purpose:** Find where to break text to fit a given pixel width; implements word-wrap.
- **Inputs:** Text buffer, pixel width, start offset, end offset of text span.
- **Outputs/Return:** `true` if end of text reached; `end_index` pointer filled with break position.
- **Side effects:** None.
- **Calls:** `GetInterfaceFont()`, `char_width()`
- **Notes:** Prefers space characters for breaks; falls back to mathematical break points based on `better_terminal_word_wrap` setting.

### handle_reading_terminal_keys
- **Signature:** `static void handle_reading_terminal_keys(short player_index, int32 action_flags)`
- **Purpose:** Process player input while reading a terminal; handle scrolling, group transitions, and special actions.
- **Inputs:** Player index, action flags (keyboard/input state).
- **Outputs/Return:** None.
- **Side effects:** Updates `terminal->current_line`, `terminal->current_group`, `terminal->phase`; may trigger teleport, sound, or light changes.
- **Calls:** `get_indexed_grouping()`, `next_terminal_group()`, `next_terminal_state()`, `teleport_to_level()`, `teleport_to_polygon()`, `play_object_sound()`, `set_tagged_light_statuses()`, `try_and_change_tagged_platform_states()`, `previous_terminal_group()`, `initialize_player_terminal_info()`
- **Notes:** Behavior varies by group type; teleport, sound, and tag groups are auto-advancing.

### MarathonTerminalCompiler::Compile
- **Signature:** `terminal_text_t* MarathonTerminalCompiler::Compile()`
- **Purpose:** Parse and compile Marathon-format terminal text (with `#logon`, `#information`, `$B` codes, etc.) into internal `terminal_text_t` structure.
- **Inputs:** Text and length passed to constructor.
- **Outputs/Return:** Pointer to compiled `terminal_text_t` (caller takes ownership).
- **Side effects:** Populates internal vectors of groupings and font changes.
- **Calls:** `FinishGroup()`, `CompileLine()`, `BuildUnfinishedGroup()`, `BuildSuccessGroup()`, `BuildFailureGroup()`, `calculate_lines_per_page()`, `calculate_maximum_lines_for_groups()`
- **Notes:** Uses Boost iostreams for streaming parse; embeds special groups (unfinished, success, failure) at compilation time.

### pack_map_terminal_data / unpack_map_terminal_data
- **Signature:** `void pack_map_terminal_data(uint8 *p, size_t count)` / `void unpack_map_terminal_data(uint8 *p, size_t count)`
- **Purpose:** Serialize/deserialize all map terminals to/from byte stream (for save files, network).
- **Inputs:** Byte pointer, count (bytes to unpack) or unused for pack.
- **Outputs/Return:** None (modifies stream in-place).
- **Side effects:** Clears `map_terminal_text` on unpack; updates stream pointer.
- **Calls:** `StreamToValue()`, `ValueToStream()`, `StreamToBytes()`, `BytesToStream()`
- **Notes:** Includes header (flags, lines_per_page, counts) and binary text; encodes sizes and grouping metadata.

## Control Flow Notes
- **Init phase:** `initialize_terminal_manager()` ΓåÆ per-player state array creation.
- **Frame update:** `update_player_for_terminal_mode()` handles timers; `update_player_keys_for_terminal()` routes input.
- **Render phase:** `_render_computer_interface()` checks dirty flag, dispatches by group type.
- **Transition phase:** Group completion triggers `next_terminal_group()` or `next_terminal_state()` or auto-aborts on special groups (teleport, sound, tag).
- **Shutdown:** `abort_terminal_mode()` exits and calls Lua `Terminal_Exit()` hook.

## External Dependencies
- **Includes:** `world.h` (points, polygons), `map.h` (saves_objects), `player.h` (player data), `screen_drawing.h` (rectangle/color/font system), `overhead_map.h` (map rendering), `lua_script.h` (scripting hooks), `sdl_fonts.h`, `joystick.h`, `Packing.h` (stream I/O)
- **External symbols used:** `dynamic_world`, `current_player_index`, `play_object_sound()`, `_render_overhead_map()`, `set_drawing_clip_rectangle()`, `_draw_screen_text()`, `get_interface_rectangle()`, `_get_interface_color()`, `draw_surface` (SDL_Surface), `world_point_to_polygon_index()`, `set_tagged_light_statuses()`, `try_and_change_tagged_platform_states()`, `L_Call_Terminal_Exit()` (Lua), `film_profile` (compatibility config)
