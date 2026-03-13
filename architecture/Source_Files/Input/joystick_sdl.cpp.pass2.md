# Source_Files/Input/joystick_sdl.cpp - Enhanced Analysis

## Architectural Role

This file is the **SDL2 gamepad abstraction layer** for the Input subsystem, bridging raw SDL controller events to the engine's scancode-based keymap pipeline and analog aim system. It handles three critical flows: (1) **device lifecycle** management via `active_instances` registry, (2) **per-frame input buffering** that decouples async SDL events from deterministic frame processing, and (3) **preference-driven input translation** that allows users to rebind axes/buttons at runtime without recompilation. The file directly feeds into both the discrete action system (via `joystick_buttons_become_keypresses()`) and the continuous aim system (via `process_joystick_axes()` ΓåÆ `process_aim_input()`).

## Key Cross-References

### Incoming (who depends on this file)
- **Event loop** (SDL handler): calls `joystick_added()`, `joystick_removed()`, `joystick_axis_moved()`, `joystick_button_pressed()` on hardware events
- **Frame logic** (main game loop): calls `joystick_buttons_become_keypresses()` and `process_joystick_axes()` each tick to consume buffered state
- **Startup/shutdown** (shell): calls `initialize_joystick()`, `enter_joystick()`, `exit_joystick()`
- **Implicit**: `input_preferences` global is read in `axis_mapped_to_action()`, `joystick_buttons_become_keypresses()`, `process_joystick_axes()`

### Outgoing (what this file depends on)
- **player.h**: `process_aim_input(flags, {dyaw, dpitch})` ΓÇö feeds analog aim deltas to player controller; `mask_in_absolute_positioning_information` macro (from `player.h` include)
- **preferences.h**: `input_preferences->key_bindings[action]`, `->controller_analog`, `->controller_sensitivity_*`, `->controller_deadzone_*`, `->controller_aim_inverted` ΓÇö all read at frame time, NOT cached
- **SDL2**: Device enumeration, controller initialization, GUID lookup for unmapped device warnings
- **Logging.h**: `logWarning()` for unmapped controller alerts

## Design Patterns & Rationale

**Async Event Buffering**: Global `axis_values[]` and `button_values[]` arrays decouple SDL's asynchronous event delivery from frame-locked processing. Events arrive whenever hardware fires; frame logic consumes them once per tick. This is idiomatic for 30-FPS-locked engine ticks.

**Instance ID Registry**: `active_instances` map tracks open SDL controller handles by instance ID, enabling device hotplug. However, **the axis/button buffers are NOT per-instance** (see Issues below).

**Preference-Driven Binding Lookup**: `axis_mapped_to_action()` queries `input_preferences->key_bindings[action]` at frame time, enabling runtime rebinding without engine restart. This avoids baking input mappings into code.

**Hybrid Input Model** (Marathon-specific): Axes are simultaneously available as (1) continuous raw values (`axis_values[axis]`) for aim processing, and (2) discrete threshold-crossing buttons (`button_values[AO_SCANCODE_BASE_JOYSTICK_AXIS_*]`) for movement. This allows the player to aim with the right stick while moving with buttons, or vice versa.

**Y-Axis Negation** (line 86): Left/right stick Y values are flipped to match Marathon's "up = negative" convention for mouselook. This is a common cross-platform quirk.

## Data Flow Through This File

```
SDL Events (async, from event loop)
  Γö£ΓöÇ joystick_added(device_index)
  Γöé   ΓööΓöÇ Validate device has SDL mapping, open SDL_GameController, store in active_instances
  Γö£ΓöÇ joystick_removed(instance_id)
  Γöé   ΓööΓöÇ Close handle, erase from active_instances
  Γö£ΓöÇ joystick_axis_moved(instance_id, axis, value)
  Γöé   ΓööΓöÇ Buffer raw value to axis_values[axis], derive threshold-crossing button states
  ΓööΓöÇ joystick_button_pressed(instance_id, button, down)
      ΓööΓöÇ Buffer button state to button_values[button]

Per-Frame Input Consumption (sync)
  Γö£ΓöÇ joystick_buttons_become_keypresses(ioKeyMap)
  Γöé   Γö£ΓöÇ Query input_preferences->controller_analog mode
  Γöé   Γö£ΓöÇ For each axis, call axis_mapped_to_action() to find which is bound to yaw/pitch
  Γöé   Γö£ΓöÇ Exclude those axes from button reporting (hybrid mode)
  Γöé   ΓööΓöÇ Write remaining button_values[] into keymap at SDL_Scancode offsets
  Γöé
  ΓööΓöÇ process_joystick_axes(flags)
      Γö£ΓöÇ For each axis in axis_mappings (hardcoded axes 2,3 ΓåÆ yaw; 8,9 ΓåÆ pitch)
      Γö£ΓöÇ Call axis_mapped_to_action(action_index) to check if axis is bound
      Γö£ΓöÇ Apply deadzone clamp (< threshold ΓåÆ 0)
      Γö£ΓöÇ Normalize to [-1, 1] and apply sensitivity curve
      Γö£ΓöÇ Convert norm to angle (768/63 radians per unit)
      Γö£ΓöÇ Apply axis negation and vertical inversion per axis_mappings
      ΓööΓöÇ Call process_aim_input(flags, {dyaw, dpitch}) to update player aim
```

## Learning Notes

**Era-appropriate design** (mid-2000s): Global input state and async event buffering were the standard approach before modern input action systems (Unreal Input System, Unity Input Manager). Decoupling events from frame logic prevents frame hitches when polling events.

**Preference coupling**: The frame-time queries to `input_preferences` mean input behavior can change mid-game (useful for UI rebinding), but also mean preference changes aren't cachedΓÇöslight performance cost for flexibility.

**Controller database reliance**: The code depends on SDL's built-in controller mapping database (`SDL_IsGameController`, `SDL_GameControllerOpen`). Unmapped controllers are rejected entirely with a warning, which is user-friendly but means custom/vintage controllers won't work without SDL updates.

**Movement vs. aiming split**: The dual-mode axis handling (can bind stick to movement buttons OR to aim, but not both simultaneously) reflects Marathon's action-game design where digital movement is primary and analog aiming is optional/secondary.

## Potential Issues

1. **Critical: Single global axis buffer** (lines 82ΓÇô89, 152ΓÇô166)  
   `axis_values[]` is global, not per-controller. If two controllers are connected, their axis data **overwrites each other**. In `process_joystick_axes()`, axis data is read without checking which controller it came from. This silently corrupts aim input in multi-controller scenarios (e.g., co-op splitscreen). The `instance_id` parameter in `joystick_axis_moved()` is **accepted but ignored**.

2. **Hardcoded axis mappings** (lines 104ΓÇô110)  
   `axis_mappings` assumes axes 2,3,8,9 map to left/right stick X/Y. SDL assigns these consistently for *recognized* GameControllers, but custom/unusual controllers might differ. No validation or fallback if an expected axis is missing.

3. **Threshold magic number** (line 90)  
   The ┬▒16384 threshold (50% of 32767 range) for button conversion is hardcoded with no explanation or configurability. This differs from typical gamepad thresholds (often 25ΓÇô30% deadzone).

4. **Incomplete instance tracking**  
   `joystick_button_pressed()` ignores `instance_id` (line 97). If multiple controllers are connected, button presses from any of them update the same global `button_values[]` array, stomping each other.

5. **Stale axis data on disconnect**  
   If a controller is removed, its last axis values remain in `axis_values[]` until overwritten by another event or the next frame. Brief visual glitch possible if disconnect happens mid-frame.
