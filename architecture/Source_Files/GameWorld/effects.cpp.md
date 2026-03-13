# Source_Files/GameWorld/effects.cpp

## File Purpose
Manages visual and audio effects in the game world (explosions, blood splashes, teleportation, media interactions, etc.). Effects are time-limited visual/audio objects that animate and clean themselves up based on definition flags and animation completion.

## Core Responsibilities
- Create and destroy effect instances at runtime
- Update effect animations and state each frame tick
- Handle special effects logic (teleportation object visibility toggling, delay handling)
- Manage effect collection loading/unloading
- Serialize and deserialize effect state for save games
- Integrate with the Lua scripting system for effect invalidation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `effect_data` | struct | Runtime effect instance (type, object ref, flags, metadata, delay); member of `EffectList` vector |
| `effect_definition` | struct | Template for an effect type (collection, shape, sound pitch, flags, delay sound); defined in effect_definitions.h |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `effect_definitions` | `effect_definition[NUMBER_OF_EFFECT_TYPES]` | static | Runtime array of effect definitions; initialized from `original_effect_definitions` |
| `EffectList` | `std::vector<effect_data>` | extern | Dynamic array of all active effects per map; indexed by effect_index |

## Key Functions / Methods

### get_effect_data
- **Signature:** `effect_data *get_effect_data(const short effect_index)`
- **Purpose:** Retrieve effect instance with bounds and usage validation
- **Inputs:** `effect_index` ΓÇö index into EffectList
- **Outputs/Return:** Pointer to effect_data struct
- **Side effects:** Asserts if index out of range or slot unused
- **Calls:** `GetMemberWithBounds()`, `SLOT_IS_USED()`
- **Notes:** Invariant enforced: slot must be marked used; provides idiot-proofing per comments

### get_effect_definition
- **Signature:** `effect_definition *get_effect_definition(const short type)`
- **Purpose:** Retrieve effect template by type
- **Inputs:** `type` ΓÇö effect type enum
- **Outputs/Return:** Pointer to effect_definition in global array
- **Calls:** `GetMemberWithBounds()`
- **Notes:** Moved to match C++ declaration order (after effect_definitions.h include)

### new_effect
- **Signature:** `short new_effect(world_point3d *origin, short polygon_index, short type, angle facing)`
- **Purpose:** Create a new effect instance in the world
- **Inputs:** `origin` (3D position), `polygon_index`, `type` (effect type), `facing` (orientation)
- **Outputs/Return:** Effect index (NONE on failure)
- **Side effects:** 
  - Creates map object (visual geometry)
  - Modifies EffectList slot (marks used, sets object_index, delay, flags)
  - Plays sound for sound-only effects
  - Sets object owner to `_object_is_effect`
- **Calls:** `get_effect_definition()`, `get_shape_animation_data()`, `play_world_sound()`, `new_map_object3d()`, `get_object_data()`, `SET_OBJECT_OWNER()`, `SET_OBJECT_INVISIBILITY()`, `SET_OBJECT_IS_MEDIA_EFFECT()`
- **Notes:** 
  - Idiot-proofs on null definition
  - Handles delay (invisible initially if delay > 0)
  - Supports sound-only effects (no visual)
  - Stores effect_index in object.permutation for reverse lookup

### update_effects
- **Signature:** `void update_effects(void)`
- **Purpose:** Update all active effects each tick; handles animation, delay countdown, and deactivation
- **Inputs:** None (iterates EffectList)
- **Outputs/Return:** None
- **Side effects:** 
  - Decrements delay counters
  - Animates effect objects
  - Removes effects on animation termination (per flags)
  - Makes twin objects visible if `_make_twin_visible` flag set
  - Calls Lua invalidation
- **Calls:** `get_object_data()`, `get_effect_definition()`, `animate_object()`, `GET_OBJECT_ANIMATION_FLAGS()`, `remove_effect()`, `SET_OBJECT_INVISIBILITY()`, `play_object_sound()`
- **Notes:** 
  - Assumed to be called once per tick (1/30 sec in Marathon)
  - Respects definition flags for when to stop (animation loop vs. transfer mode loop)
  - Delay mechanism allows effects to remain invisible/silent initially before activation

### remove_effect
- **Signature:** `void remove_effect(short effect_index)`
- **Purpose:** Deactivate and clean up an effect instance
- **Inputs:** `effect_index` ΓÇö index of effect to remove
- **Outputs/Return:** None
- **Side effects:** 
  - Removes associated map object (visual)
  - Marks slot as free in EffectList
  - Invalidates effect for Lua scripts
- **Calls:** `get_effect_data()`, `remove_map_object()`, `L_Invalidate_Effect()`, `MARK_SLOT_AS_FREE()`
- **Notes:** Called when animation completes or on explicit cleanup

### remove_all_nonpersistent_effects
- **Signature:** `void remove_all_nonpersistent_effects(void)`
- **Purpose:** Clear all effects flagged to end at animation loop (cleanup on level change)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `remove_effect()` on matching effects
- **Calls:** `get_effect_definition()`, `remove_effect()`
- **Notes:** Idiot-proofs on null definition; used for map transitions

### teleport_object_out / teleport_object_in
- **Signature:** 
  - `void teleport_object_out(short object_index)`
  - `void teleport_object_in(short object_index)`
- **Purpose:** Coordinate teleportation effect with object visibility (out: hide object, create effect; in: make object visible, create effect)
- **Inputs:** `object_index` ΓÇö object being teleported
- **Outputs/Return:** None
- **Side effects:** 
  - Creates teleport effect (`_effect_teleport_object_out` or `_effect_teleport_object_in`)
  - Hides/shows object
  - Stores object_index in effect.data for cross-reference
  - Syncs effect object shape/sequence/transfer mode with real object
  - Plays teleport sound
- **Calls:** `get_object_data()`, `new_effect()`, `get_effect_data()`, `SET_OBJECT_INVISIBILITY()`, `play_object_sound()`, `Sound_TeleportOut()`
- **Notes:** 
  - `teleport_object_out()` makes object invisible, starts fold-out effect
  - `teleport_object_in()` checks if already teleporting in (prevents duplication), creates fold-in effect
  - Effect definition flags `_make_twin_visible` triggers `SET_OBJECT_INVISIBILITY(false)` when effect ends

### unpack_effect_data / pack_effect_data
- **Signature:** 
  - `uint8 *unpack_effect_data(uint8 *Stream, effect_data *Objects, size_t Count)`
  - `uint8 *pack_effect_data(uint8 *Stream, effect_data *Objects, size_t Count)`
- **Purpose:** Serialize/deserialize effect instances to/from byte stream
- **Inputs:** Stream pointer, effect array, count
- **Outputs/Return:** Advanced stream pointer
- **Side effects:** Reads/writes 32 bytes per effect
- **Calls:** `StreamToValue()`, `ValueToStream()`
- **Notes:** Packs type, object_index, flags, data, delay (14 bytes used, 11 padding words for alignment)

### unpack_effect_definition / pack_effect_definition
- **Signature:** Multiple overloads (with/without explicit array, M1 legacy format)
- **Purpose:** Serialize effect definitions
- **Inputs:** Stream, count; optional explicit definition array
- **Outputs/Return:** Advanced stream pointer
- **Side effects:** Global `effect_definitions` modified on unpack
- **Calls:** `StreamToValue()`, `ValueToStream()`
- **Notes:** `unpack_m1_effect_definition()` handles Marathon 1 format (no sound_pitch or delay_sound)

### init_effect_definitions
- **Signature:** `void init_effect_definitions()`
- **Purpose:** Reset effect definitions to original/default state
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Copies from `original_effect_definitions` to `effect_definitions`
- **Calls:** `memcpy()`
- **Notes:** Called at level init or reset

## Control Flow Notes
- **Per-Frame**: `update_effects()` called during world update loop; processes all EffectList entries, handles animation and cleanup
- **Creation**: `new_effect()` typically called by gameplay code (explosions, item pickups, projectile impacts)
- **Cleanup**: Effects self-remove when animation/transfer mode completes, or removed explicitly on map exit
- **Special**: Teleportation uses dual effect+object coordination to synchronize visibility and visual morphing

## External Dependencies
- **map.h**: World geometry, object structures (`object_data`, `polygon_data`), map object functions (`new_map_object3d()`, `remove_map_object()`)
- **interface.h**: Shape/collection info (`get_shape_animation_data()`, `mark_collection_for_loading()`)
- **SoundManager.h**: Sound playback (`play_world_sound()`, `play_object_sound()`, `Sound_TeleportOut()`)
- **lua_script.h**: Lua integration (`L_Invalidate_Effect()`)
- **Packing.h**: Binary I/O utilities (`StreamToValue()`, `ValueToStream()`)
- **effect_definitions.h**: Effect template data and flags (included for definitions)
- **world.h**: 3D geometry types (`world_point3d`, `angle`)
