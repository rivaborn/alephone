# Source_Files/GameWorld/lightsource.cpp - Enhanced Analysis

## Architectural Role

This file implements the **dynamic lighting subsystem** within GameWorld, providing time-animated brightness values consumed by the rendering pipeline every frame. It's a relatively isolated module that bridges trigger/device control (via tagging) and rendering needs. Unlike entity simulation (monsters, projectiles) which requires collision/AI integration, lightsource.cpp operates independently: lights don't collide with the world, don't pathfind, and don't damage entities. Instead, they're consumed by RenderMain during visibility and shading passes to illuminate geometry and sprites. The subsystem's core innovation is the **six-phase state machine with randomized animation parameters**, enabling lights to transition smoothly between on/off while executing preconfigured animation curves (constant, linear, smooth, flicker, random, fluorescent).

## Key Cross-References

### Incoming (who depends on this file)
- **RenderMain** (`render.cpp`, `OGL_Render.cpp`, `OGL_Textures.cpp`) ΓåÆ calls `get_light_intensity()` per-frame to fetch brightness for polygon/sprite shading during rasterization
- **GameWorld main loop** (`marathon2.cpp` `update_world()`) ΓåÆ calls `update_lights()` each 30 FPS tick to advance animation phases and recalculate intensities
- **Device/Platform triggers** (`devices.cpp`, `platforms.cpp`, likely polygon-based triggers) ΓåÆ calls `set_light_status()` and `set_tagged_light_statuses()` to toggle light activation
- **Lua scripting** (`lua_script.h` `L_Call_Light_Activated()`) ΓåÆ receives callbacks when lights transition states (hook for map scripts)
- **Map loading/save** (`game_wad.cpp`) ΓåÆ calls pack/unpack functions to serialize light state

### Outgoing (what this file depends on)
- **map.h** (`MAXIMUM_LIGHTS_PER_MAP`, slot macros `SLOT_IS_USED/MARK_SLOT_AS_USED`, `global_random()`, structure offsets)
- **lightsource.h** (header; light_data, static_light_data, lighting_function_specification definitions)
- **Packing.h** (`StreamToValue()`, `ValueToStream()` for binary serialization)
- **lua_script.h** (scripting event hook)
- **cseries.h** (core types: `_fixed`, `uint8`; macros: `TEST_FLAG16`, `SET_FLAG16`; utilities: `csprintf()`, `vhalt()`, `GetMemberWithBounds()`)
- **Trig tables** (external `cosine_table`, `TRIG_MAGNITUDE`, `TRIG_SHIFT`, `HALF_CIRCLE`)

## Design Patterns & Rationale

**State Machine (six-phase FSM):** Lights don't toggle instantly; they transition through states `becoming_active ΓåÆ primary_active ΓåÆ secondary_active ΓåÆ (loop or becoming_inactive)`. This allows smooth fade-in/out animations and prevents visual pops. The design rationale (inferable from state names and animation function specs) is to enable:
- Smooth transitions: "becoming" states use fade curves
- Stable lit state: "primary" states hold steady (or pulse)
- Potential flickering: "secondary" states can add variation without full shutdown

**Animation Dispatch Table:** Six lighting functions (constant, linear, smooth, flicker, random, fluorescent) are registered in a function pointer array and selected per-state via `lighting_functions[function_index]`. This is a classic procedural dispatch pattern from pre-OOP game engines, avoiding virtual function overhead and allowing easy extension without subclassing.

**Template-based Instantiation:** Three `light_definition` structs (normal, strobe, lava) predefine all 6 state animation specs. `new_light()` clones a template into runtime `light_data`, making light types pluggable without code changes (supports MML/XML-driven customization via external config not visible here).

**Slot-based Object Pool:** Lights are stored in a fixed-size vector with slot reuse (via `SLOT_IS_FREE`, `MARK_SLOT_AS_USED`), avoiding malloc/free per-frame and enabling deterministic memory layoutΓÇöcritical for networked replay sync.

## Data Flow Through This File

```
Creation Phase:
  new_light(static_light_data)
    ΓåÆ find free slot in LightList
    ΓåÆ initialize: phase=0, state=secondary (not primary), mark used
    ΓåÆ call change_light_state(secondary) to sample random period/intensity
    ΓåÆ call change_light_state(primary) to transition to steady state
    ΓåÆ compute initial intensity via lighting_function_dispatch()

Per-Frame Update (each 30 FPS tick):
  update_lights()
    ΓåÆ for each used light:
        light->phase += 1
        rephase_light(light_index)
          ΓåÆ while phase >= period: subtract period, transition to next state
          ΓåÆ change_light_state() resamples period/intensity delta
        light->intensity = lighting_function_dispatch(
          function_type, initial_intensity, final_intensity, phase, period)

Rendering Phase:
  renderer calls get_light_intensity(light_index)
    ΓåÆ return light->intensity (already computed by update_lights)

Interaction Phase:
  set_light_status(light_index, new_status) [from trigger/device/Lua]
    ΓåÆ detect status change
    ΓåÆ call change_light_state() to initiate "becoming" transition
    ΓåÆ trigger Lua hook L_Call_Light_Activated()

```

**Key insight:** Light intensity is computed once per frame in `update_lights()`, then retrieved read-only in the rendering pass. This decouples simulation from rendering frame rates (supports > 30 FPS rendering with 30 FPS ticks).

## Learning Notes

- **Fixed-point arithmetic throughout** (`_fixed` type): Ensures deterministic computation across platforms and deterministic replay in multiplayer. The smooth_lighting_proc uses cosine tables with bit shifts (`>> TRIG_SHIFT`) rather than floating-pointΓÇötypical of 1990s game engines before hardware FPUs were universal.

- **Deterministic randomness:** `global_random()` is a seeded PRNG (not `rand()`). Each frame tick uses the same seed, so lights animate identically across replay/network sync. However, RNG seed management is opaque (defined elsewhere in `world.cpp`).

- **No dynamic allocation:** Lights are pre-allocated in a vector; reused via slot macros. This is a pool-based pattern for predictable performance and no memory fragmentationΓÇöcrucial for games targeting 30 FPS on 1990s hardware.

- **Serialization first-class:** Pack/unpack functions for every structure suggest save-game persistence was a priority (Marathon 1 saved mid-mission). Modern engines would use reflection/serialization frameworks; this is explicit handwritten code.

- **Legacy format conversion:** `convert_old_light_data_to_new()` converts Marathon 1 light definitions to Marathon 2, showing upgrade support. The rationale: maintain backwards compatibility with user maps.

- **Scripting integration late-added:** The Lua hook `L_Call_Light_Activated()` in `set_light_status()` is a post-hoc extension point, allowing custom map scripts to react to lights without modifying the core system.

## Potential Issues

1. **Infinite loop in `rephase_light()` if period = 0:** The loop `while (phase >= light->period)` assumes `period > 0`. Comments mention "functions with zero periods are skipped," but no runtime guard prevents a zero period from causing infinite loops.

2. **Function dispatch bounds unchecked at runtime:** `lighting_function_dispatch()` uses `assert(function_index >= 0 && function_index < NUMBER_OF_LIGHTING_FUNCTIONS)` for debug builds only. A corrupted `light_data.state` could yield invalid function index, dereferencing a garbage pointer in release builds.

3. **RNG state not serialized:** `change_light_state()` calls `global_random()` to vary period/intensity, but if RNG state isn't saved during map serialization, replayed games will have non-deterministic light animations despite fixed-point math.

4. **Null function pointer dereference risk in `change_light_state()`:** `get_lighting_function_specification()` could theoretically return null if `light->state` is corrupted; dereferencing `function->period` without null check would crash.

5. **Integer overflow in period calculation:** `light->period = function->period + global_random() % (function->delta_period + 1)` could overflow if period values are near max short range and delta is large (no clamping).
