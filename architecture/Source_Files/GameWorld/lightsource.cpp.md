# Source_Files/GameWorld/lightsource.cpp

## File Purpose
Implements the dynamic lighting system for the Marathon game engine (Aleph One). Manages light creation, state transitions, animation curves, and intensity calculations. Supports six-phase state machines, multiple animation function types (constant, linear, smooth, flicker, random, fluorescent), and legacy Marathon 1 light data conversion.

## Core Responsibilities
- Create and manage dynamic light instances (`new_light()`)
- Animate lights via phased state machines (6 states: becoming/primary/secondary active/inactive)
- Calculate light intensities each frame based on parameterized animation curves
- Handle state transitions when animation periods expire (`rephase_light()`)
- Query and modify light activation status (`get_light_status()`, `set_light_status()`)
- Support tagged light groups for trigger-based activation
- Serialize/deserialize light data for save games and map loading
- Convert Marathon 1 legacy light data to Marathon 2 format

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `light_data` | struct | Runtime dynamic light state: flags, current state, phase, period, intensity, animation parameters |
| `static_light_data` | struct | Immutable light template: type, flags, 6 lighting function specs (one per state), tag |
| `lighting_function_specification` | struct | Animation curve parameters: function type, period, delta_period, intensity, delta_intensity |
| `light_definition` | struct | Predefined light template with sounds and default static_light_data |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `LightList` | `vector<light_data>` | global | All active lights in current map; converted to `lights` pointer macro |
| `light_definitions[3]` | `light_definition[]` | static | Preconfigured templates: _normal_light, _strobe_light, _lava_light |
| `lighting_functions[6]` | function pointer array | static | Dispatch table for 6 animation curve implementations |
| `old_light_definitions[8]` | `static_light_data[]` | static | Marathon 1 light conversion templates |

## Key Functions / Methods

### new_light
- **Signature:** `short new_light(struct static_light_data *data)`
- **Purpose:** Create and initialize a new light instance from static configuration
- **Inputs:** `data` ΓÇö light template with animation specs
- **Outputs/Return:** Light index (0ΓÇôMAXIMUM_LIGHTS_PER_MAP), or `NONE` if no free slot
- **Side effects:** Marks slot as used; initializes phase, period, state, and intensity; calls `change_light_state()` twice and `L_Call_Light_Activated()`
- **Calls:** `change_light_state()`, `rephase_light()`, `lighting_function_dispatch()`, `LIGHT_IS_INITIALLY_ACTIVE()` macro
- **Notes:** Idiot-proofs null `data`; sets initial intensity to 0, transitions through secondary states first

### update_lights
- **Signature:** `void update_lights(void)`
- **Purpose:** Advance all active lights by one frame
- **Inputs:** None (reads global `LightList`)
- **Outputs/Return:** None
- **Side effects:** Increments phase for each light; calls `rephase_light()` and `lighting_function_dispatch()` to recalculate intensity
- **Calls:** `rephase_light()`, `lighting_function_dispatch()`, `get_lighting_function_specification()`
- **Notes:** Called each game tick; skips unused slots

### change_light_state
- **Signature:** `void change_light_state(size_t light_index, short new_state)`
- **Purpose:** Transition a light to a new state and initialize animation parameters
- **Inputs:** `light_index` ΓÇö light to modify; `new_state` ΓÇö target state (one of 6)
- **Outputs/Return:** None
- **Side effects:** Sets `phase` to 0; samples random period and intensity delta; stores initial/final intensity; updates state
- **Calls:** `get_light_data()`, `get_lighting_function_specification()`, `global_random()`
- **Notes:** Adds randomness to period and final intensity; called during state transitions in `rephase_light()` and when toggling via `set_light_status()`

### rephase_light
- **Signature:** `static void rephase_light(short light_index)`
- **Purpose:** Handle phase wrapping and state transitions when a light's period expires
- **Inputs:** `light_index` ΓÇö light to advance
- **Outputs/Return:** None
- **Side effects:** Loop: subtracts period from phase; calls `change_light_state()` with next state; sets final `phase` to wrapped value
- **Calls:** `get_light_data()`, `change_light_state()`, `LIGHT_IS_STATELESS()` macro
- **Notes:** Implements state machine: becoming_activeΓåÆprimary_activeΓåÆsecondary_activeΓåÆ(loop or inactive); stateless lights skip inactive states

### get_light_status / set_light_status
- **Signature:** `bool get_light_status(size_t light_index)` / `bool set_light_status(size_t light_index, bool new_status)`
- **Purpose:** Query or modify light on/off state
- **Inputs:** `light_index`, `new_status` (for set)
- **Outputs/Return:** Current status (bool)
- **Side effects:** (set only) Calls `change_light_state()`, Lua hook `L_Call_Light_Activated()`, panel assumption update
- **Calls:** `get_light_data()`, `change_light_state()`, `L_Call_Light_Activated()`, `assume_correct_switch_position()`, `LIGHT_IS_STATELESS()` macro
- **Notes:** State codes collapse into active vs. inactive; stateless lights skip transitions

### set_tagged_light_statuses
- **Signature:** `bool set_tagged_light_statuses(short tag, bool new_status)`
- **Purpose:** Activate/deactivate all lights with matching tag
- **Inputs:** `tag`, `new_status`
- **Outputs/Return:** `true` if any light changed
- **Side effects:** Calls `set_light_status()` for each matching light
- **Notes:** Used by tag switches; `tag == 0` is a no-op

### get_light_intensity
- **Signature:** `_fixed get_light_intensity(size_t light_index)`
- **Purpose:** Retrieve current brightness value for rendering
- **Inputs:** `light_index`
- **Outputs/Return:** Fixed-point intensity (0 = black, FIXED_ONE = full)
- **Side effects:** None
- **Notes:** Fallback returns 0 (blackness) if light invalid

### Lighting function implementations (constant, linear, smooth, flicker, random, fluorescent)
- **Signature:** `static _fixed XXX_lighting_proc(_fixed initial_intensity, _fixed final_intensity, short phase, short period)`
- **Purpose:** Compute intensity at given phase using specific animation curve
- **Inputs:** `initial_intensity`, `final_intensity` (start/end), `phase` (0 to period), `period` (cycle length in ticks)
- **Outputs/Return:** Interpolated _fixed intensity
- **Notes:** 
  - *constant*: returns `final_intensity` regardless of phase
  - *linear*: piecewise linear interpolation
  - *smooth*: cosine-based smooth interpolation
  - *flicker*: smooth curve plus random jitter
  - *random*: uniform random in [initial, final] or [final, initial]
  - *fluorescent*: random 50/50 between initial and final

### Pack/unpack functions
- **Patterns:** `unpack_old_light_data()`, `pack_old_light_data()`, `unpack_static_light_data()`, `pack_static_light_data()`, `unpack_light_data()`, `pack_light_data()`
- **Purpose:** Serialize/deserialize light data to/from byte streams for save files and map loading
- **Notes:** Assertions verify byte count matches expected struct size; helper `StreamToLightSpec()` / `LightSpecToStream()` for function specs

### convert_old_light_data_to_new
- **Signature:** `void convert_old_light_data_to_new(static_light_data* NewLights, old_light_data* OldLights, int Count)`
- **Purpose:** Migrate Marathon 1 light data to Marathon 2 format
- **Inputs:** Old and new light arrays, count
- **Outputs/Return:** None (in-place conversion)
- **Side effects:** Copies old light definitions; patches intensities and periods based on old light type/mode
- **Notes:** Special handling for strobe lights (period scaling); maps old modes to initially_active flag

## Control Flow Notes

**Initialization:**
- `new_light()` is called when loading map or placing lights; initializes phase to 0 and transitions through two state changes to reach steady state

**Per-frame update:**
- `update_lights()` is called each tick; increments phase, detects period overflow via `rephase_light()`, and recalculates intensity

**State transitions:**
- `rephase_light()` implements a 6-state machine: becoming_active ΓåÆ primary_active ΓåÆ secondary_active ΓåÆ (becoming_inactive or loop), with randomized period/intensity
- External triggers call `set_light_status()` or `set_tagged_light_statuses()` to initiate transitions (not inferable as part of frame loop, but called from polygon triggers)

**Rendering:**
- `get_light_intensity()` is called by renderer to fetch current brightness; intensity is pre-computed each frame by `update_lights()`

## External Dependencies

- `map.h` ΓÇö Map structures, MAXIMUM_LIGHTS_PER_MAP, slot macros (SLOT_IS_USED, MARK_SLOT_AS_USED), global_random()
- `lightsource.h` ΓÇö Header (light_data, static_light_data, lighting_function_specification definitions, constants, prototypes)
- `Packing.h` ΓÇö StreamToValue(), ValueToStream() for byte serialization
- `lua_script.h` ΓÇö L_Call_Light_Activated() hook
- `cseries.h` ΓÇö Core types (_fixed, uint8), macros (TEST_FLAG16, SET_FLAG16), utilities (csprintf, vhalt)

**Defined elsewhere (not in this file):**
- `cosine_table`, `TRIG_MAGNITUDE`, `TRIG_SHIFT`, `HALF_CIRCLE` (trig tables, math)
- `GetMemberWithBounds()` (bounds-checking accessor helper)
- `assume_correct_switch_position()` (panel/device state update)
- `obj_copy()` (struct assignment helper)
