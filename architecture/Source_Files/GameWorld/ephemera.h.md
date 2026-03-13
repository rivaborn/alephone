# Source_Files/GameWorld/ephemera.h

## File Purpose
Defines the interface for managing ephemeral objectsΓÇötemporary visual effects and animated sprites that exist in the game world for limited durations. Ephemera are lightweight objects stored in polygons, with automatic cleanup based on animation state.

## Core Responsibilities
- Allocate and initialize ephemera storage pools per level
- Create ephemera at world positions with visual representation (shape descriptor)
- Manage ephemera lifecycle (creation, removal, garbage collection)
- Maintain polygon-based spatial indexing of ephemera
- Update ephemera animation state each frame
- Track which polygons were rendered to optimize cleanup

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `object_data` | struct (from map.h) | Used to store ephemera visual/animation state; owner flags reused for ephemera control |
| `shape_descriptor` | typedef | 16-bit descriptor (collection + shape index) for rendering ephemera graphics |
| `world_point3d` | struct (from world.h) | 3D world position for ephemera placement |

## Global / File-Static State
None visible from header. Implementation (ephemera.cpp) manages:
- Ephemera pool array
- Per-polygon ephemera linked lists
- Animation and removal tracking

## Key Functions / Methods

### allocate_ephemera_storage
- Signature: `void allocate_ephemera_storage(int max_ephemera)`
- Purpose: Pre-allocate memory for the ephemera pool at startup
- Inputs: `max_ephemera` ΓÇô maximum number of concurrent ephemera
- Outputs/Return: None
- Side effects: Allocates dynamic memory; must be called once before level loads
- Calls: (not inferable)
- Notes: Called during initialization before entering levels

### init_ephemera
- Signature: `void init_ephemera(int16_t polygon_count)`
- Purpose: Reset ephemera state for a new level; initialize per-polygon ephemera lists
- Inputs: `polygon_count` ΓÇô number of polygons in the level
- Outputs/Return: None
- Side effects: Clears all ephemera; resets polygon indices
- Calls: (not inferable)
- Notes: Called when entering a level

### new_ephemera
- Signature: `int16_t new_ephemera(const world_point3d& origin, int16_t polygon_index, shape_descriptor shape, angle facing)`
- Purpose: Create a new ephemera object at a location
- Inputs: 3D position, polygon context, visual shape, initial facing angle
- Outputs/Return: Index of new ephemera (`-1` or NONE if pool exhausted, inferred)
- Side effects: Inserts into polygon's ephemera list; allocates from pool
- Calls: (not inferable)
- Notes: Facing angle may be used for directional sprites or ignored for omnidirectional effects

### remove_ephemera
- Signature: `void remove_ephemera(int16_t ephemera_index)`
- Purpose: Destroy an ephemera and free its slot
- Inputs: Ephemera index
- Outputs/Return: None
- Side effects: Removes from polygon's linked list; marks slot as free
- Calls: (not inferable)

### get_ephemera_data
- Signature: `object_data* get_ephemera_data(int16_t ephemera_index)`
- Purpose: Retrieve the object_data structure for a given ephemera (for animation/rendering access)
- Inputs: Ephemera index
- Outputs/Return: Pointer to object_data, or null if invalid index
- Side effects: None
- Calls: (not inferable)

### get_polygon_ephemera
- Signature: `int16_t get_polygon_ephemera(int16_t polygon_index)`
- Purpose: Get the head of the ephemera linked list in a polygon
- Inputs: Polygon index
- Outputs/Return: First ephemera index in polygon, or NONE if empty
- Side effects: None
- Calls: (not inferable)
- Notes: Caller iterates via `object_data->next_object` field

### remove_ephemera_from_polygon / add_ephemera_to_polygon
- Signatures: 
  - `void remove_ephemera_from_polygon(int16_t ephemera_index)`
  - `void add_ephemera_to_polygon(int16_t ephemera_index, int16_t polygon_index)`
- Purpose: Manually move ephemera between polygon lists
- Inputs: Ephemera index; (for add) target polygon
- Outputs/Return: None
- Side effects: Updates polygon lists and ephemera's polygon field
- Calls: (not inferable)
- Notes: Used when ephemera moves across polygon boundaries during animation

### set_ephemera_shape
- Signature: `void set_ephemera_shape(int16_t ephemera_index, shape_descriptor shape)`
- Purpose: Change the visual representation of an ephemera mid-animation
- Inputs: Ephemera index, new shape descriptor
- Outputs/Return: None
- Side effects: Updates object_data shape field; may reset animation frame
- Calls: (not inferable)

### note_ephemera_polygon_rendered
- Signature: `void note_ephemera_polygon_rendered(int16_t polygon_index)`
- Purpose: Notify the ephemera system that a polygon's rendering pass completed
- Inputs: Polygon index
- Outputs/Return: None
- Side effects: Marks polygon as rendered; allows cleanup of ephemera in unrendered polygons
- Calls: (not inferable)
- Notes: Called post-render to optimize garbage collection of off-screen ephemera

### update_ephemera
- Signature: `void update_ephemera()`
- Purpose: Advance all ephemera animations by one tick; remove finished ephemera
- Inputs: None (uses global state)
- Outputs/Return: None
- Side effects: Updates animation frames; checks `_ephemera_end_when_animation_loops` flag; removes expired ephemera
- Calls: (not inferable; likely calls animate_object and remove_ephemera internally)
- Notes: Called once per game tick in the main update loop

## Control Flow Notes
**Ephemera lifecycle:**
1. **Level init**: `init_ephemera()` clears all ephemera at map entry.
2. **Gameplay**: Monsters, weapons, or the engine call `new_ephemera()` to spawn effects (e.g., blood, sparks, dust).
3. **Per-frame**: `update_ephemera()` advances animation and checks termination condition (`_ephemera_end_when_animation_loops`).
4. **Rendering**: Renderer iterates polygon ephemera lists (via `get_polygon_ephemera()`) and calls `note_ephemera_polygon_rendered()` post-pass.
5. **Cleanup**: Ephemera are removed via `remove_ephemera()` when animation loops (if flag set) or manually.

## External Dependencies
- **map.h**: `object_data` (stores ephemera state), `polygon_data` (spatial structure)
- **shape_descriptors.h**: `shape_descriptor` (visual representation)
- **world.h**: `world_point3d`, `angle` (position and orientation types)
