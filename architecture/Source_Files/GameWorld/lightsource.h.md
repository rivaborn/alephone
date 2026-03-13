# Source_Files/GameWorld/lightsource.h

## File Purpose
Defines light source data structures, lighting function specifications, and state management for the game engine's dynamic lighting system. Provides interfaces for creating, updating, and querying lights, with backward compatibility for Marathon I map format.

## Core Responsibilities
- Define static light data structures with type and behavior flags
- Specify lighting intensity transition functions (constant, linear, smooth, flicker, random, fluorescent)
- Define dynamic light state tracking (active/inactive states, current intensity, phase)
- Manage global light list as a dynamic array
- Provide light status queries and state mutation operations
- Support serialization/deserialization of light data
- Maintain Marathon I format compatibility layer for legacy map support

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| lighting_function_specification | struct | Defines how light intensity transitions: function type, period, and intensity deltas (14 bytes) |
| static_light_data | struct | Immutable light properties: type, flags, phase, six lighting function specs, tag (100 bytes) |
| light_data | struct | Dynamic light state: flags, state enum, current intensity, phase/period, initial/final intensities plus static data (128 bytes) |
| old_light_data | struct | Marathon I format compatibility; includes mode, min/max intensity, period (32 bytes) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| LightList | std::vector\<light_data\> | global extern | All active lights in current map |
| lights | macro (LightList.data()) | global | Array pointer for C-style iteration |
| MAXIMUM_LIGHTS_PER_MAP | macro (LightList.size()) | global | Dynamic light count |

## Key Functions / Methods

### new_light
- Signature: `short new_light(struct static_light_data *data)`
- Purpose: Create and initialize a new light from static specification
- Inputs: static_light_data containing type, flags, behavior functions
- Outputs/Return: Light index (short)
- Side effects: Adds entry to global LightList

### update_lights
- Signature: `void update_lights(void)`
- Purpose: Step all active lights' intensity transitions each frame
- Inputs: None (operates on global LightList)
- Outputs/Return: None (modifies LightList entries in-place)
- Side effects: Updates intensity, phase, state for all lights based on lighting functions

### get_light_status / set_light_status
- Purpose: Query or change light active/inactive state
- Inputs: light_index (size_t), optional new active state (bool)
- Outputs/Return: Current/new state (bool)
- Side effects: Modifies light_data.state

### set_tagged_light_statuses
- Purpose: Batch state change for lights sharing a tag (e.g., linked switches)
- Inputs: tag (short), new_status (bool)
- Outputs/Return: bool
- Side effects: Updates state for all lights matching tag

### get_light_intensity
- Signature: `_fixed get_light_intensity(size_t light_index)`
- Purpose: Accessor for current computed intensity
- Outputs/Return: Fixed-point intensity value

### Packing/Unpacking Functions
- `unpack_*/pack_*` (old_light_data, static_light_data, light_data variants)
- Purpose: Serialize/deserialize light structures to/from byte streams
- Inputs: Byte stream pointer, object array, count
- Outputs/Return: Updated stream pointer (allows chaining)

### convert_old_light_data_to_new
- Purpose: Convert Marathon I light format to current format during map load
- Inputs: old_light_data array, target static_light_data array, count
- Side effects: Populates NewLights array

## Control Flow Notes
Header is declarative; actual update loop calls `update_lights()` each frame. Lights progress through six states: becoming_active ΓåÆ primary_active ΓåÆ becoming_inactive ΓåÆ primary_inactive (or secondary variants). Each state uses its own lighting function spec to transition intensity over time. Lights may start out-of-phase (phase field) to stagger effects.

## External Dependencies
- `cstypes.h`: Provides `_fixed` (16.16 fixed-point), `int16`, `uint16`, `uint8` types
- `<vector>`: C++ standard library for dynamic light array
- Not inferable: Implementation of lighting functions, state machine logic (defined elsewhere in .c file)
