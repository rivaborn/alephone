# Source_Files/GameWorld/scenery.cpp

## File Purpose
Manages scenery objectsΓÇöstatic or animated decorative/environmental world elements. Handles creation, animation updates, destruction effects, and configuration via Marathon Map Language (MML) definitions.

## Core Responsibilities
- Create scenery objects with proper flags and physics properties
- Maintain and update animations for dynamic scenery each game tick
- Apply damage to scenery, including destruction transitions and effects
- Load and apply MML-based scenery definition overrides
- Query scenery dimensions, collections, and animation properties

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `scenery_definition` | struct | Defines properties of a scenery type (shape, flags, destruction effects, dimensions). Defined in scenery_definitions.h; not fully visible here. |
| `AnimatedSceneryObjects` | vector\<short\> | Growable list of object indices tracking which scenery objects have active animations. |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AnimatedSceneryObjects` | `vector<short>` | static | Indices of scenery objects currently animating; updated each frame. |
| `ok_to_reset_scenery_solidity` | bool | static | Flag allowing solidity reset when MML modifies scenery definitions post-initialization. |
| `original_scenery_definitions` | `scenery_definition*` | static | Backup of original definitions for MML reset; allocated on first MML parse. |

## Key Functions / Methods

### initialize_scenery
- **Signature:** `void initialize_scenery(void)`
- **Purpose:** Initialize scenery subsystem at startup/level load.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Reserves capacity in `AnimatedSceneryObjects` vector (32 elements).
- **Calls:** None (just vector operation).
- **Notes:** Called early in game/level init to pre-allocate vector memory.

### new_scenery
- **Signature:** `short new_scenery(object_location *location, short scenery_type)`
- **Purpose:** Create a new scenery object at a location.
- **Inputs:** `location` ΓÇô world position and polygon; `scenery_type` ΓÇô index into scenery definitions.
- **Outputs/Return:** Object index on success, `NONE` on failure (invalid type or map object creation failed).
- **Side effects:** Allocates a map object; sets owner to `_object_is_scenery`, solidity flag from definition, permutation to scenery type.
- **Calls:** `get_scenery_definition()`, `new_map_object()`, `get_object_data()`, `SET_OBJECT_OWNER()`, `SET_OBJECT_SOLIDITY()`.
- **Notes:** Returns `NONE` if scenery_type is out of range. Does not add to `AnimatedSceneryObjects` here; that happens on randomization.

### animate_scenery
- **Signature:** `void animate_scenery(void)`
- **Purpose:** Update animation for all active scenery each game tick.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `animate_object()` on each object in `AnimatedSceneryObjects`.
- **Calls:** `animate_object()` per animated scenery.
- **Notes:** Assumes ╬öt = 1 tick.

### deanimate_scenery
- **Signature:** `void deanimate_scenery(short object_index)`
- **Purpose:** Remove a scenery object from the animated list.
- **Inputs:** `object_index` ΓÇô index of object to stop animating.
- **Outputs/Return:** None
- **Side effects:** Finds and erases object_index from `AnimatedSceneryObjects` if present.
- **Calls:** Standard vector iterator operations.
- **Notes:** Linear search through vector; safe if object not in list.

### randomize_scenery_shape
- **Signature:** `void randomize_scenery_shape(short object_index)`
- **Purpose:** Randomize animation sequence for a single scenery object.
- **Inputs:** `object_index` ΓÇô scenery object to randomize.
- **Outputs/Return:** None
- **Side effects:** If `randomize_object_sequence()` returns false (object is animating), adds object_index to `AnimatedSceneryObjects`.
- **Calls:** `get_object_data()`, `get_scenery_definition()`, `randomize_object_sequence()`.
- **Notes:** Early return if definition is invalid.

### randomize_scenery_shapes
- **Signature:** `void randomize_scenery_shapes(void)`
- **Purpose:** Randomize animation for all scenery objects on the map.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears `AnimatedSceneryObjects`; iterates all object slots, finds scenery, randomizes and re-populates the animated list.
- **Calls:** `get_scenery_definition()`, `randomize_object_sequence()` per scenery object.
- **Notes:** Typically called when loading a level.

### get_scenery_dimensions
- **Signature:** `void get_scenery_dimensions(short scenery_type, world_distance *radius, world_distance *height)`
- **Purpose:** Retrieve bounding dimensions for a scenery type.
- **Inputs:** `scenery_type`, output pointers `radius`, `height`.
- **Outputs/Return:** Fills `*radius` and `*height`; defaults to 0 if definition invalid.
- **Side effects:** None (read-only).
- **Calls:** `get_scenery_definition()`.

### damage_scenery
- **Signature:** `void damage_scenery(short object_index)`
- **Purpose:** Apply damage to scenery; if destroyable, transition to destroyed shape and play effect.
- **Inputs:** `object_index` ΓÇô scenery object being damaged.
- **Outputs/Return:** None
- **Side effects:** Changes object shape to destroyed_shape if `_scenery_can_be_destroyed` flag set; resets sequence; optionally randomizes destroyed animation; creates destruction effect; changes owner to `_object_is_normal`.
- **Calls:** `get_object_data()`, `get_scenery_definition()`, `randomize_object_sequence()` (conditional), `new_effect()` (if effect != NONE).
- **Notes:** Obeys `film_profile.fix_destroy_scenery_random_frame` to optionally randomize destroyed frame.

### get_scenery_collection
- **Signature:** `bool get_scenery_collection(short scenery_type, short& collection)`
- **Purpose:** Retrieve the collection index for a scenery's normal shape.
- **Inputs:** `scenery_type`, output reference `collection`.
- **Outputs/Return:** true if valid; false otherwise. Fills `collection` on success.
- **Side effects:** None (read-only).
- **Calls:** `get_scenery_definition()`, `GET_DESCRIPTOR_COLLECTION()`.

### get_damaged_scenery_collection
- **Signature:** `bool get_damaged_scenery_collection(short scenery_type, short& collection)`
- **Purpose:** Retrieve collection for destroyed shape if scenery is destroyable.
- **Inputs:** `scenery_type`, output reference `collection`.
- **Outputs/Return:** true if valid and destroyable; false otherwise.
- **Side effects:** None (read-only).
- **Calls:** `get_scenery_definition()`, flag check, `GET_DESCRIPTOR_COLLECTION()`.

### get_scenery_definition (private)
- **Signature:** `scenery_definition* get_scenery_definition(short scenery_type)`
- **Purpose:** Look up scenery definition with bounds checking.
- **Inputs:** `scenery_type`
- **Outputs/Return:** Pointer to definition, or NULL if out of range.
- **Side effects:** None.
- **Calls:** `GetMemberWithBounds()` (from external library).

### reset_scenery_solidity (private)
- **Signature:** `static void reset_scenery_solidity()`
- **Purpose:** Reapply solidity flags to all scenery objects after MML changes.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Iterates all object slots; updates solidity flag for scenery per current definitions if `ok_to_reset_scenery_solidity` is true.
- **Calls:** `get_scenery_definition()`, `SET_OBJECT_SOLIDITY()`.
- **Notes:** Workaround for MML loading after initial scenery creation.

### reset_mml_scenery
- **Signature:** `void reset_mml_scenery()`
- **Purpose:** Restore original scenery definitions, discarding MML modifications.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Copies `original_scenery_definitions` back to `scenery_definitions`; frees backup.
- **Calls:** Standard C memory functions.

### parse_mml_scenery
- **Signature:** `void parse_mml_scenery(const InfoTree& root)`
- **Purpose:** Parse MML "object" elements to override scenery definitions.
- **Inputs:** `root` ΓÇô InfoTree node containing "object" children.
- **Outputs/Return:** None
- **Side effects:** On first call, backs up original definitions. Reads flags, radius, height, destruction effect, and shape descriptors from MML; updates `scenery_definitions` array. Calls `reset_scenery_solidity()` at end.
- **Calls:** `InfoTree::children_named()`, `read_indexed()`, `read_attr()`, `read_shape()`, `reset_scenery_solidity()`, malloc/free.
- **Notes:** Assumes MML indices are valid; includes both "normal" and "destroyed" shape child nodes.

---

## Control Flow Notes
- **Level Init:** `initialize_scenery()` ΓåÆ scenery objects created via `new_scenery()` ΓåÆ `randomize_scenery_shapes()` populates animated list.
- **Game Loop (each tick):** `animate_scenery()` updates all animated scenery.
- **Damage:** `damage_scenery()` called by external code; transitions destroyed scenery and fires effects.
- **MML:** `parse_mml_scenery()` called at config time; modifies definitions and resets solidity on existing objects.

## External Dependencies
- **map.h:** `new_map_object()`, `get_object_data()`, `animate_object()`, `randomize_object_sequence()`, `MAXIMUM_OBJECTS_PER_MAP`, object owner/flag macros.
- **effects.h:** `new_effect()`.
- **scenery.h:** Header declaring public interface (not shown).
- **InfoTree.h:** XML configuration parsing.
- Indirect: **dynamic_limits.h** (via get_dynamic_limit, not directly visible).
- **scenery_definitions.h:** External array `scenery_definitions` and constant `NUMBER_OF_SCENERY_DEFINITIONS`.
