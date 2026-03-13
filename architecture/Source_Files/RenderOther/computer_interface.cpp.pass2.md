# Source_Files/RenderOther/computer_interface.cpp - Enhanced Analysis

## Architectural Role

This file is a **world-facing UI system**, not merely a display overlayΓÇöit bridges the rendering pipeline with GameWorld simulation by triggering teleports, sound playback, platform/light activation, and Lua hooks. Terminal interactions are persistent per-player (separate state for multiplayer), serialized in save files, and integrated with the device activation system (`devices.cpp`), making terminals first-class interactive world objects that happen to render text-rich content.

## Key Cross-References

### Incoming (who depends on this file)
- **GameWorld/devices.cpp** ΓÇö Triggers terminal entry via `enter_computer_interface()` when player activates a switch or panel object
- **GameWorld/marathon2.cpp** ΓÇö Main loop calls `update_player_for_terminal_mode()` and `_render_computer_interface()` each frame
- **GameWorld/player.cpp** ΓÇö Player state holds `player_terminal_data` array; terminal mode blocks regular movement input
- **Lua/lua_script.cpp** ΓÇö Hooks `Terminal_Exit()` callback when terminal closes
- **Network subsystem** ΓÇö `pack/unpack_map_terminal_data()` serializes terminals for save files and network sync
- **Input/Input handling** ΓÇö Routes keyboard/joystick events via `handle_reading_terminal_keys()` during terminal mode

### Outgoing (what this file depends on)
- **RenderOther/screen_drawing.cpp** ΓÇö Calls `_draw_screen_text()`, `_draw_screen_shape()`, `_draw_screen_shape_centered()`, `_get_interface_color()`, font/color accessors
- **RenderOther/overhead_map.cpp** ΓÇö Calls `_render_overhead_map()` for checkpoint visualization
- **GameWorld/map.cpp** ΓÇö Calls `world_point_to_polygon_index()` for spatial checkpoint queries; reads `dynamic_world` geometry
- **GameWorld/sound/** ΓÇö Calls `play_object_sound()` for sound groups; activates group via tag IDs
- **GameWorld/lightsource.cpp** ΓÇö Calls `set_tagged_light_statuses()` to activate lights triggered by terminal tags
- **GameWorld/platforms.cpp** ΓÇö Calls `try_and_change_tagged_platform_states()` for platform activation (doors, elevators)
- **Files/Packing.h** ΓÇö Binary serialization via `StreamToValue()`, `ValueToStream()`, `BytesToStream()`, `StreamToBytes()`
- **Lua** ΓÇö Calls `L_Call_Terminal_Exit()` on terminal mode exit
- **Shell/interface.cpp** ΓÇö Reads `film_profile.better_terminal_word_wrap` compatibility setting

## Design Patterns & Rationale

**1. State Machine + Rich Document Format**
- Terminal content is compiled into a hierarchical structure: groups ΓåÆ text spans ΓåÆ font changes (style/color at byte offsets)
- Marathon text format (`#logon`, `#information`, `$B` style codes) is compiled to internal binary representation
- Per-player state tracks position within the document (current_group, current_line, phase)
- Special groups (unfinished/success/failure/teleport/sound/tag/camera) auto-transition or trigger side effectsΓÇöthis is semantic dispatch, not just rendering

**2. Graceful Degradation with Null Checks**
- Nearly all accessors (`get_indexed_terminal_data()`, `get_indexed_grouping()`, `get_indexed_font_changes()`) return NULL on bounds errors rather than asserting
- Entry point `enter_computer_interface()` plays a fallback sound if no terminal data exists
- Rendering functions skip over missing content (e.g., PICT load failure doesn't crash, just skips image)
- This suggests robustness against map corruption or custom WAD incompleteness

**3. Clipping + Rendering Context Stacking**
- File explicitly mentions clipping rectangle manipulation; sets terminal-specific clip region before rendering, restores after
- Coordinates with main rendering pipeline's `set_drawing_clip_rectangle()` to avoid trampling global state
- Suggests careful context management to coexist with HUD, overhead map, and main viewport rendering

**4. Text Layout as Deterministic Algorithm**
- `calculate_line()` implements word-wrap with branching logic for `film_profile.better_terminal_word_wrap`
- Honors space-based breaks and empirically-determined break characters (`can_break_after()`)
- This is a **compatibility shim**: original Marathon had simpler/buggier word-wrap; new code provides better breaks but remains configurable for replay determinism

**5. Per-Player State Isolation**
- Terminal state is per-player (`player_terminals` array), not global
- Enables multiplayer where each player navigates independently or multiple screens show same terminal at different positions
- Packed/unpacked during save/network sync as part of player state

## Data Flow Through This File

```
Game World (Map)
  Γåô
[Player activates switch/panel] ΓåÉ devices.cpp
  Γåô
enter_computer_interface(player_id, terminal_id, completion_flag)
  Γö£ΓåÆ get_indexed_terminal_data(terminal_id) ΓåÉ from map_terminal_text[] (loaded by game_wad.cpp)
  Γö£ΓåÆ calculate_lines_per_page() [font metrics]
  Γö£ΓåÆ next_terminal_group() [select first visible group]
  ΓööΓåÆ play_object_sound() [if no terminal data]
  Γåô
[Per-frame loop]
  Γö£ΓåÆ update_player_for_terminal_mode() [timer/timeout logic]
  Γö£ΓåÆ handle_reading_terminal_keys() [player input processing]
  Γöé  Γö£ΓåÆ next_terminal_group() / next_terminal_state() [group transitions]
  Γöé  ΓööΓåÆ [Dispatch by group type]
  Γöé     Γö£ΓåÆ teleport_to_level() / teleport_to_polygon() ΓåÉ GameWorld manipulation
  Γöé     Γö£ΓåÆ play_object_sound() ΓåÉ Audio subsystem
  Γöé     Γö£ΓåÆ set_tagged_light_statuses() ΓåÉ lightsource activation
  Γöé     Γö£ΓåÆ try_and_change_tagged_platform_states() ΓåÉ device activation
  Γöé     ΓööΓåÆ L_Call_Terminal_Exit() ΓåÉ Lua hook
  ΓööΓåÆ _render_computer_interface() [if dirty flag set]
     Γö£ΓåÆ Dispatch by group->type
     Γö£ΓåÆ draw_logon_text() / draw_computer_text() / present_checkpoint_text() / display_picture_with_text() / fill_terminal_with_static()
     ΓööΓåÆ [Core rendering]
        Γö£ΓåÆ _draw_screen_text() ΓåÉ screen_drawing.cpp
        Γö£ΓåÆ _draw_screen_shape() ΓåÉ shape rendering (PICT resources)
        Γö£ΓåÆ _render_overhead_map() ΓåÉ spatial visualization
        ΓööΓåÆ _fill_rect / _frame_rect ΓåÉ borders/backgrounds
```

## Learning Notes

- **Terminal system as event trigger**: Terminals are not passive UI; they're active world devices that mutate GameWorld state (teleport, activate lights/platforms). The `handle_reading_terminal_keys()` dispatch is closer to an event dispatcher than a UI controller.
- **Marathon format archaeology**: The presence of `MarathonTerminalCompiler` parsing `#logon`/`#information` codes, plus the special group auto-insertion (unfinished/success/failure), reveals this codebase is emulating a decades-old text format for replay/mod compatibility.
- **Encoding layer**: `encode_text()` / `decode_text()` suggest a second-level serialization beyond the Marathon text formatΓÇöpossibly a compression or binary-compatible format used in actual WAD storage.
- **Double rendering passes**: `draw_terminal_borders()` is called *after* other terminal stuff to fix overdraw bugs (per changelog). This is a depth-ordering workaround, suggesting the file learned about layering issues through empirical debugging.
- **Compatibility mode as explicit toggle**: `film_profile.better_terminal_word_wrap` is a settings knob, not a version check. This suggests the game supports mixed-era mods and needs deterministic playback.

## Potential Issues

- **Text encoding assumption**: If `encode_text()` uses a legacy or undocumented format, future maintainers may struggle with text corruption if the encoding logic is lost or modified.
- **Spatial query for checkpoints**: `find_checkpoint_location()` performs polygon lookups; if the map is corrupted or checkpoint index is out-of-bounds, this could silently fail (graceful degradation) or render a blank map, confusing players.
- **Input handling state machine complexity**: The dispatcher for group types (teleport/sound/tag auto-advance) is implicit in the `handle_reading_terminal_keys()` function. Adding a new group type requires understanding this hidden state machine.
- **Clipping context fragility**: If a render call within `_render_computer_interface()` fails to properly restore clipping before returning, subsequent HUD or main viewport rendering may be clipped incorrectly. The manual push/pop pattern is error-prone without RAII.
