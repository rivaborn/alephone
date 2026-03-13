# Source_Files/GameWorld/lightsource.h - Enhanced Analysis

## Architectural Role

Lightsource is a **dynamic world entity subsystem** within GameWorld responsible for animating light intensities across six state phases. It integrates tightly with the main game loop (marathon2.cpp calls `update_lights()` each frame) and the serialization pipeline (game_wad.cpp packs/unpacks light data during map I/O). The system is designed to decouple light behavior specification (static_light_data with six lighting_function_specification instances) from runtime state (light_data), enabling efficient batch updates and seamless save/load cycles. Marathon I format support suggests this represents a major engine overhaul where lighting transitioned from simple modes (on/off/flicker) to a generalized function-based animation system.

## Key Cross-References

### Incoming (who depends on this file)
- **GameWorld/marathon2.cpp**: Calls `update_lights()` in main game loop; triggers level transitions
- **GameWorld/devices.cpp**: Calls `change_light_state()` when switches/terminals toggle light status (via `set_light_status`, `set_tagged_light_statuses`)
- **Files/game_wad.cpp**: Calls `pack_light_data` / `unpack_light_data` during map load/save; also `convert_old_light_data_to_new` for Marathon I maps
- **Rendering subsystem**: Reads `LightList` and `get_light_intensity()` to determine dynamic lighting contribution
- **RenderMain/render.cpp** (inferred): Queries current light intensities for polygons during visibility/rasterization

### Outgoing (what this file depends on)
- **CSeries/cstypes.h**: Fixed-point (_fixed) and integer types; flag macros (TEST_FLAG16, SET_FLAG16)
- **std::vector** (C++): LightList manages dynamic allocation of light_data structures
- **No direct Lua/Scripting dependency**: Event callbacks likely triggered by `change_light_state`, not scripted directly

## Design Patterns & Rationale

**State Machine + Lighting Functions Pattern**
- Six distinct states (becoming_active, primary_active, secondary_active, becoming_inactive, primary_inactive, secondary_inactive) allow smooth transitions with distinct behavior in each phase
- Each state has its own `lighting_function_specification`, enabling different animation curves during activation vs. deactivation
- This dual-intensity (primary/secondary) design permits flickering/pulsing without requiring separate light entities

**Why structured this way:**
- Separates **static configuration** (what lights *look like*) from **dynamic state** (what they *currently are*)ΓÇöenabling efficient batching and deterministic serialization
- Six lighting functions (constant, linear, smooth, flicker, random, fluorescent) are premixed rather than procedural, reducing per-frame computation
- Phase field (initial timing offset) avoids synchronized flickering across multiple lightsΓÇöimproves perceived randomness/naturalism

**Legacy Compatibility Layer**
- old_light_data and conversion function allow Marathon I maps (simpler mode-based system) to coexist with Marathon II generalized system
- Conversion is one-directional (oldΓåÆnew); no back-conversion, signaling the old system is deprecated but required for archive support

**Vector-based Light List**
- Comment mentions "took over their maximum number as how many of them"ΓÇödynamic allocation replaced fixed MAXIMUM_LIGHTS_PER_MAP
- Macro aliases (lights, MAXIMUM_LIGHTS_PER_MAP) provide C-style array API over std::vector for backward compatibility with existing code

## Data Flow Through This File

**Load Phase (Map Initialization)**
- game_wad.cpp unserializes light_data from WAD or converts old_light_data
- `new_light(static_light_data*)` adds to LightList
- Each light initialized with phase (stagger start times) and state flags

**Per-Frame Update**
- main game loop calls `update_lights()`
- For each light in LightList:
  - Apply lighting function based on current state, phase, and period
  - Compute new intensity (result of function evaluation)
  - Possibly transition state (e.g., becoming_active ΓåÆ primary_active after timeout)
  - Store updated intensity for rendering queries

**State Mutation (Switches/Devices)**
- `set_light_status()` or `set_tagged_light_statuses()` called from devices.cpp
- Initiates state transition (e.g., primary_active ΓåÆ becoming_inactive)
- Triggering a new lighting function spec for smooth fade-out

**Serialization (Save/Restore)**
- Pack/unpack functions traverse LightList, serializing light_data byte-by-byte
- Maintains 128-byte structure size for binary compatibility
- game_wad.cpp chains these calls to serialize full map state

## Learning Notes

**Idiomatic to This Era (1995ΓÇô2001, Pre-Shader)**
- **Fixed-point arithmetic** instead of floats (more deterministic across platforms; required on PowerPC Macs)
- **Baked animation tables** (six predefined lighting functions) vs. procedural or shader-driven lightingΓÇöreflects CPU constraints
- **Phase-based state machines** for deterministic replay and network synchronization (mentioned in git comments: "Jul 1, 2000" updates for packing routines)
- **Binary serialization with explicit struct sizes** (const SIZEOF_light_data = 128) to enforce layout; modern engines use reflection or JSON

**Modern Engines Differ:**
- Would use data-driven animation curves (keyframes, splines) instead of hardcoded function types
- Would compute lighting per-pixel in shaders, not as a global intensity
- Would use float for broader range; fixed-point is an obsolete performance optimization
- Would decouple entity definition from serialization format (e.g., Lua/YAML config ΓåÆ runtime representation)

## Potential Issues

1. **No bounds checking on light_index** in accessors: `get_light_status()`, `set_light_status()`, `get_light_intensity()` accept `size_t light_index` but file provides no size validation. Caller must ensure index < MAXIMUM_LIGHTS_PER_MAP. (Defensiveness depends on calling conventions in marathon2.cpp and devices.cpp.)

2. **Phase wraparound undefined**: `phase` field and `period` interact to drive animation, but file does not document modulo behavior or overflow handling. Lighting function implementation (in .cpp) is the authoritative definition.

3. **Old light data loss on conversion**: `convert_old_light_data_to_new()` is one-way; no reverse conversion. Marathon II features (secondary intensity, complex function specs) cannot round-trip through a Marathon I save file. This is intentional but worth documenting.
