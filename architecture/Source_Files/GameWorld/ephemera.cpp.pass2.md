# Source_Files/GameWorld/ephemera.cpp - Enhanced Analysis

## Architectural Role

Ephemera implements the "fire-and-forget" visual effects subsystem within GameWorldΓÇöshort-lived, non-interactive objects (explosions, splashes, teleport effects) that animate once and self-destruct. It bridges the entity lifecycle system (allocation, animation, removal) with the spatial rendering pipeline (per-polygon organization). Unlike persistent entities (monsters, items, player), ephemera require no AI, physics, or persistence logic, making them ideal candidates for aggressive pooling and auto-cleanup.

## Key Cross-References

### Incoming (who depends on this file)
- **marathon2.cpp** ΓÇö Main game loop calls `update_ephemera()` in frame tick; various event handlers (projectile hit, platform activation, damage) call `new_ephemera()`
- **Rendering system** ΓÇö RenderPlaceObjs queries `get_polygon_ephemera()` to collect visible ephemera for depth sorting; shapes/animation data drives visual output
- **Lua scripting** ΓÇö Scripts can trigger `new_ephemera()` via bindings; `L_Invalidate_Ephemera()` callback notifies Lua of removals (maintains any Lua object references)
- **Map system** ΓÇö `object_data` pool reuses animation/sequence fields from shared entity structure; `animate_object()` applies per-frame animation state

### Outgoing (what this file depends on)
- **map.h** ΓÇö `object_data` structure definition; `animate_object()` function for frame advancement; `local_random()` for animation state seeding; macros (`MARK_SLOT_AS_USED`, `MARK_SLOT_AS_FREE`, `BUILD_SEQUENCE`, `NONE` sentinel)
- **interface.h** ΓÇö `get_shape_animation_data()` retrieves animation metadata (frame count, view count, animation type)
- **lua_script.h** ΓÇö `L_Invalidate_Ephemera()` callback for cleanup notifications
- **dynamic_limits.h** ΓÇö `get_dynamic_limit(_dynamic_limit_ephemera)` provides pool size cap from config

## Design Patterns & Rationale

**Object Pool with Intrusive Linked-List Free-List** ΓÇö Pre-allocated fixed-size vector; free slots chained via `next_object` field without separate list nodes. Rationale: Ephemera spawn and die rapidly (e.g., explosion spawns 10+ particles); dynamic allocation would thrash the heap. Pool provides O(1) alloc/free without fragmentation.

**Spatial Binning by Polygon** ΓÇö Per-polygon head pointers in `polygon_ephemera`; ephemera form linked lists per polygon. Rationale: Aligns with rendering visibility culling (only visible polygons need animation); enables fast spatial queries (e.g., "which effects are here?").

**Safe Removal During Iteration** ΓÇö `update_ephemera()` snapshots `next_index` before calling `remove_ephemera()`, avoiding iterator invalidation. Elegant sentinel-based design vs. deferred-delete queues.

**Lazy Shape Animation Initialization** ΓÇö `set_ephemera_shape()` initializes sequence/transfer_mode based on animation metadata; branches on M1 vs. M2+ shape semantics. Rationale: Bridges legacy content compatibility while allowing modern animation features (transfer modes, timing).

## Data Flow Through This File

**Creation path:**
- Caller (projectile, platform, Lua) ΓåÆ `new_ephemera(location, polygon, shape, angle)` 
- Allocate unused slot from pool ΓåÆ Initialize object_data (location, polygon, flags, shape)
- Call `set_ephemera_shape()` ΓåÆ Query animation metadata, randomize start frame
- Call `add_ephemera_to_polygon()` ΓåÆ Prepend to polygon's linked list
- Return index to caller

**Update path:**
- Each frame: `update_ephemera()` iterates all polygons ΓåÆ for each ephemera in polygon list
- Call `animate_object()` ΓåÆ Increments animation frame; sets `_obj_last_frame_animated` flag when final frame reached
- Check if both `_obj_last_frame_animated` and `_ephemera_end_when_animation_loops` flags set ΓåÆ auto-remove
- Renderer reads via `get_polygon_ephemera()` to collect rendering primitives

**Removal path:**
- Remove from polygon list via linear search (finds predecessor, unlinks)
- Call `L_Invalidate_Ephemera()` ΓåÆ Lua cleanup
- Return to pool free-list

## Learning Notes

- **Pooling trade-off:** Fixed pre-allocation (must set `_dynamic_limit_ephemera` correctly) vs. dynamic allocation overhead. Developers using this would learn to profile pool exhaustion in effect-heavy levels.
- **Reuse of shared animation infrastructure:** Ephemera piggyback `object_data` and `animate_object()` rather than reimplementing animation. Shows how Marathon consolidates disparate entity types under unified animation system.
- **M1 vs. M2+ branching:** `shapes_file_is_m1()` check reveals engine heritageΓÇöM1 shapes were static; M2+ added multi-view animated shapes. Ephemera must handle both gracefully.
- **Idiomatic sentinel value:** `NONE` (-1) as list terminator, free-slot indicator is a common 1990s pattern; modern engines use optional types or indices.
- **Polygon-centric world model:** Unlike modern spatial hashing, this engine organizes everything (entities, lights, effects) into polygon-linked lists, reflecting the 2.5D BSP-tree world representation.

## Potential Issues

- **Linear-search removal:** `remove_ephemera_from_polygon()` walks the linked list to find predecessorΓÇöO(n) in worst case. A polygon with hundreds of active effects could stall per-frame updates. (Singly-linked list limitation; would need doubly-linked or generation counters to fix.)
- **Silent animation failure:** If `get_shape_animation_data(shape)` returns nullptr (malformed shape collection), `set_ephemera_shape()` returns early without logging. Ephemera silently animate at frame 0 foreverΓÇöhard to debug if shape asset is corrupted.
- **No animation clipping:** If animation frame advances beyond `frames_per_view`, behavior depends on `animate_object()` implementation; wrapping vs. clamping not locally visible.
