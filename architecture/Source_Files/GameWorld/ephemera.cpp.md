# Source_Files/GameWorld/ephemera.cpp

## File Purpose
Manages ephemeraΓÇöshort-lived visual objects (effects, particles) in the game world. Implements an object pool allocator for efficient reuse and organizes ephemera spatially by polygon for fast updates and lookups.

## Core Responsibilities
- Object pool allocation/deallocation (linked-list free-list design)
- Create and initialize ephemera with shape, location, and animation state
- Remove ephemera and invalidate Lua references
- Maintain per-polygon linked lists of ephemera for spatial queries
- Animate ephemera each frame and auto-remove when animation loops
- Track shape animation data and transfer modes

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ObjectDataPool` | class | Pool allocator for `object_data` instances; manages free/used slots as linked list |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `polygon_ephemera` | `std::vector<int16_t>` | global | Head pointer to ephemera list for each polygon; NONE if empty |
| `ephemera_pool` | `ObjectDataPool` | static | Singleton pool managing all ephemera object allocations |

## Key Functions / Methods

### allocate_ephemera_storage
- Signature: `void allocate_ephemera_storage(int max_ephemera)`
- Purpose: Pre-allocate the object pool to a fixed size
- Inputs: `max_ephemera` ΓÇö max concurrent ephemera
- Outputs/Return: None
- Side effects: Resizes `ephemera_pool` vector
- Calls: `ephemera_pool.resize()`
- Notes: Called during engine initialization; size comes from `_dynamic_limit_ephemera`

### init_ephemera
- Signature: `void init_ephemera(int16_t polygon_count)`
- Purpose: Initialize ephemera system for a new level; set up per-polygon lists and pool state
- Inputs: `polygon_count` ΓÇö number of polygons in the map
- Outputs/Return: None
- Side effects: Clears and resizes `polygon_ephemera` vector; initializes pool as free-list
- Calls: `polygon_ephemera.clear()`, `polygon_ephemera.resize()`, `ephemera_pool.init()`
- Notes: Must be called before any ephemera are created

### new_ephemera
- Signature: `int16_t new_ephemera(const world_point3d& location, int16_t polygon_index, shape_descriptor shape, angle facing)`
- Purpose: Create and initialize a new ephemera object at the given location
- Inputs: `location` (3D world position), `polygon_index`, `shape` descriptor, `facing` angle
- Outputs/Return: Ephemera index (NONE if pool exhausted)
- Side effects: Allocates from pool; sets object fields (polygon, location, facing, sequence, shape); inserts into polygon's ephemera list; calls `set_ephemera_shape()`
- Calls: `ephemera_pool.get_unused()`, `ephemera_pool.get()`, `set_ephemera_shape()`, `add_ephemera_to_polygon()`, `MARK_SLOT_AS_USED`
- Notes: Initializes `parasitic_object` to NONE; flags set to `0x8000` (SLOT_IS_USED); returns NONE if polygon_index is NONE or pool is full

### remove_ephemera
- Signature: `void remove_ephemera(int16_t ephemera_index)`
- Purpose: Remove an ephemera, unlink from polygon, and return to pool
- Inputs: `ephemera_index` ΓÇö index to remove
- Outputs/Return: None
- Side effects: Invalidates Lua references; removes from polygon list; returns slot to free-list
- Calls: `remove_ephemera_from_polygon()`, `L_Invalidate_Ephemera()`, `ephemera_pool.release()`
- Notes: Called when animation completes or explicitly destroyed

### get_ephemera_data
- Signature: `object_data* get_ephemera_data(int16_t ephemera_index)`
- Purpose: Retrieve pointer to ephemera object data
- Inputs: `ephemera_index`
- Outputs/Return: Pointer to `object_data` in pool
- Side effects: None
- Calls: `ephemera_pool.get()`

### get_polygon_ephemera
- Signature: `int16_t get_polygon_ephemera(int16_t polygon_index)`
- Purpose: Retrieve head of ephemera list for a polygon
- Inputs: `polygon_index`
- Outputs/Return: Index of first ephemera in polygon (NONE if none)
- Side effects: None
- Calls: None (direct vector access)

### remove_ephemera_from_polygon
- Signature: `void remove_ephemera_from_polygon(int16_t ephemera_index)`
- Purpose: Unlink ephemera from its polygon's linked list
- Inputs: `ephemera_index`
- Outputs/Return: None
- Side effects: Modifies polygon list pointers and object's `polygon` field
- Calls: `ephemera_pool.get()`, pointer traversal through `next_object` chain
- Notes: Linear search through list to find predecessor; sets `object.polygon = NONE`

### add_ephemera_to_polygon
- Signature: `void add_ephemera_to_polygon(int16_t ephemera_index, int16_t polygon_index)`
- Purpose: Insert ephemera at head of a polygon's ephemera list
- Inputs: `ephemera_index`, `polygon_index`
- Outputs/Return: None
- Side effects: Updates `polygon_ephemera[polygon_index]` head pointer; sets object's `next_object` and `polygon` fields
- Calls: `get_ephemera_data()`
- Notes: O(1) insertion at head; typical linked-list prepend

### set_ephemera_shape
- Signature: `void set_ephemera_shape(int16_t ephemera_index, shape_descriptor shape)`
- Purpose: Assign shape and configure animation/sequence state based on animation data
- Inputs: `ephemera_index`, `shape` descriptor
- Outputs/Return: None
- Side effects: Sets object's `shape`, `sequence`, `transfer_mode`, `transfer_phase` fields
- Calls: `get_ephemera_data()`, `get_shape_animation_data()`, `shapes_file_is_m1()`, `local_random()`, `BUILD_SEQUENCE`
- Notes: Distinguishes between M1 static vs. animated shapes; randomizes start frame for unanimated; sets xfer_normal for animated

### update_ephemera
- Signature: `void update_ephemera()`
- Purpose: Main per-frame update: animate all ephemera and remove those whose animation has looped
- Inputs: None
- Outputs/Return: None
- Side effects: Calls `animate_object()` on each ephemera; removes objects when both `_obj_last_frame_animated` and `_ephemera_end_when_animation_loops` flags are set
- Calls: `animate_object()`, `remove_ephemera()` (during iteration with safe next-index caching)
- Notes: Safe removal during iteration via `next_index` snapshot

### ObjectDataPool::init
- Signature: `void ObjectDataPool::init()`
- Purpose: Initialize the pool as a linked-list free-list of all slots
- Inputs: None
- Outputs/Return: None
- Side effects: Chains all slots via `next_object` field; sets `first_unused_` to 0
- Calls: None (direct member access)
- Notes: Handles empty pool (sets `first_unused_` = NONE)

### ObjectDataPool::get_unused
- Signature: `int16_t ObjectDataPool::get_unused()`
- Purpose: Allocate the next free slot from the pool
- Inputs: None
- Outputs/Return: Index of allocated slot (NONE if pool full)
- Side effects: Unlinks from free-list; marks slot as used
- Calls: `MARK_SLOT_AS_USED`
- Notes: O(1) allocation; maintains `first_unused_` pointer

### ObjectDataPool::release
- Signature: `void ObjectDataPool::release(int16_t index)`
- Purpose: Return a slot to the free-list
- Inputs: `index` ΓÇö slot to release
- Outputs/Return: None
- Side effects: Marks slot as free; prepends to free-list
- Calls: `MARK_SLOT_AS_FREE`
- Notes: O(1) deallocation

## Control Flow Notes
- **Init phase:** Engine calls `allocate_ephemera_storage()` then `init_ephemera()` per-level.
- **Per-frame:** Game loop calls `update_ephemera()`, which animates all ephemera and removes those reaching animation end.
- **Creation:** Caller invokes `new_ephemera()`, which allocates, initializes, and inserts into polygon list.
- **Removal:** Manual `remove_ephemera()` or automatic during `update_ephemera()` when animation completes.

## External Dependencies
- **dynamic_limits.h** ΓÇö `get_dynamic_limit(_dynamic_limit_ephemera)` for pool size
- **interface.h** ΓÇö `get_shape_animation_data()` to fetch animation metadata
- **lua_script.h** ΓÇö `L_Invalidate_Ephemera()` to notify Lua of removals
- **map.h** ΓÇö `object_data` structure; `animate_object()`, `local_random()`, macros (`MARK_SLOT_AS_FREE`, `MARK_SLOT_AS_USED`, `BUILD_SEQUENCE`, `NONE`)
- **shapes_file_is_m1()** ΓÇö (defined elsewhere) branching for Marathon 1 vs. 2+ shape behavior
