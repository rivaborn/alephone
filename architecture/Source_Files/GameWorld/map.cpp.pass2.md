# Source_Files/GameWorld/map.cpp - Enhanced Analysis

## Architectural Role

Map.cpp functions as the **spatial and lifecycle hub** of the game engine, serving as the central nervous system connecting geometry, entities, and cross-engine queries. It bridges three critical concerns: (1) **spatial organization** via polygon-linked object lists enabling O(polygon_size) entity queries instead of O(all_entities), (2) **entity lifecycle management** from creation through deferred cleanup, and (3) **physics-agnostic world queries** (line-of-sight, sound obstruction, collision detection) that don't require rendering or physics subsystems. The file acts as a fa├ºade: physics.cpp calls `keep_line_segment_out_of_walls()` for movement; render.cpp calls `get_polygon_data()` for visibility; sound system calls `line_is_obstructed()` for propagation; monsters.cpp calls `translate_map_object()` for pathfinding-based movement.

## Key Cross-References

### Incoming (who depends on this file)

- **RenderMain/render.cpp**: `allocate_render_memory()` paired with `allocate_map_memory()`; `build_render_tree()` traverses `PolygonList` for visibility
- **GameWorld/physics.cpp**: `keep_line_segment_out_of_walls()` called per-frame for collision response; `find_line_crossed_leaving_polygon()` for boundary detection
- **GameWorld/monsters.cpp**: `translate_map_object()` for AI-driven movement; `activate_nearby_monsters()` scans polygon object lists
- **GameWorld/projectiles.cpp**: `translate_map_object()` for ballistics; collision queries via polygon adjacency
- **GameWorld/platforms.cpp**: `changed_polygon()` called when platform geometry changes; cascades to adjacent polygons
- **Sound/SoundManager**: `line_is_obstructed()` checked per-frame for 3D audio propagation and muffling
- **Lua/lua_map.cpp**: Annotation functions, object creation/deletion with `L_Invalidate_Object()` callbacks
- **Files/game_wad.cpp**: Loads level data into `PolygonList`, `LineList`, etc.; calls `initialize_map_for_new_level()`

### Outgoing (what this file depends on)

- **RenderMain/shape_descriptors.h**: `GET_DESCRIPTOR_COLLECTION()` macro in `mark_map_collections()` for texture resolution
- **GameWorld/media.h**: `get_media_data()` for sound muffling; liquid height queries in movement
- **GameWorld/lightsource.h**: Indirect dependency via platform activation triggering light state changes
- **GameWorld/flood_map.h**: Polygon graph built once, used by pathfinding for monster navigation
- **Sound/SoundManager.h**: `SoundManager::instance()->PlaySound()` in `play_*_sound()` family
- **CSeries/cseries.h**: Memory allocation, `obj_clear()`, `GetMemberWithBounds()` bounds validation

## Design Patterns & Rationale

1. **Polygon-Based Spatial Hashing** (O(1) spatial lookup)
   - Each `polygon_data` maintains a linked list of objects (`first_object` ΓåÆ chain of `next_object` indices). Avoids global O(n) iteration for "which entities are in polygon X?" queries. Essential for AI line-of-sight checks, collision detection, and sound propagation.

2. **Deferred Operation Queue** (`sDeferredObjectListInsertions`)
   - Insertions scheduled but not executed until `perform_deferred_polygon_object_list_manipulations()`. Enables network prediction rollback: speculate entity positions, defer list updates, then replay if prediction is wrong. Modern netcode pattern adapted to this engine.

3. **Dual-World Architecture** (static_world vs dynamic_world)
   - Separates immutable level properties (music, mission type, geometry) from mutable game state (entity counts, scores, tick progression). Allows fast level reset: call `obj_clear(*dynamic_world)` preserving static_world and music references.

4. **Progressive Modernization Trail**
   - Fixed-size arrays (`struct object_data *objects`) replaced by STL vectors (`vector<object_data> ObjectList`), but old pointer-based code remains (calling `GetMemberWithBounds(objects, ...)` where `objects` is a macro alias). Indicates multi-year refactoring: code is mid-migration from fixed to dynamic allocation.

5. **Garbage Collection via Owner Flags** (`turn_object_to_shit()`)
   - Objects marked `_object_is_garbage` rather than immediately freed. Enforces per-polygon and per-map garbage limits, preventing visual clutter from corpses/ammunition. Cleanup is deferred and quota-based, not immediate.

6. **Cascading Geometry Propagation** (`changed_polygon()`)
   - Platform height changes call `changed_polygon()` which propagates through adjacent polygons via `find_adjacent_polygon()`, updating collision geometry. No immediate re-triangulation; deferred until next query.

## Data Flow Through This File

**Level Initialization Pipeline:**
```
game_wad.cpp loads WAD
  ΓåÆ allocate_map_memory() allocates static_world, dynamic_world pointers
  ΓåÆ File I/O populates PolygonList, LineList, SideList, EndpointList
  ΓåÆ initialize_map_for_new_level() clears dynamic state, preserves game progress
  ΓåÆ mark_map_collections(true) scans geometry, queues texture loads
```

**Per-Frame Entity Movement:**
```
physics.cpp: accelerate_player() computes desired new position
  ΓåÆ keep_line_segment_out_of_walls() clips movement, returns adjusted heights
  ΓåÆ translate_map_object() moves entity: removes from old polygon, adds to new
    ΓåÆ Updates parasitic objects (attached effects, rider objects)
    ΓåÆ Maintains polygon.first_object linked-list invariant
```

**Sound Propagation (per-frame ambient and reactive sounds):**
```
SoundManager: enqueue ambient sound sources in polygon
  ΓåÆ line_is_obstructed(polygon1, pt1, polygon2, pt2) checks path
    ΓåÆ Iterates polygon boundaries via find_line_crossed_leaving_polygon()
    ΓåÆ Detects media boundary crossings (water/lava obstruction)
    ΓåÆ Returns obstruction flags
  ΓåÆ _sound_obstructed_proc() categorizes into wall/media/muffling modes
  ΓåÆ SoundManager applies attenuation and transfer mode
```

**Object Cleanup (deferred garbage collection):**
```
Projectile detonates / Monster dies
  ΓåÆ turn_object_to_shit(object_index) marks owner=_object_is_garbage
  ΓåÆ If quota exceeded, calls remove_map_object() on oldest garbage
  ΓåÆ Unlinks from polygon.first_object, invalidates Lua references
  ΓåÆ Slot marked SLOT_AS_FREE for reuse
```

## Learning Notes

1. **Polygon Graph as First-Class Citizen**: This engine treats the polygon BSP as the data structure, not just rendering artifact. Pathfinding, spatial queries, sound propagation, and collision all pivot around polygon adjacency. Modern engines use spatial hashes or octrees; Marathon uses polygon topologyΓÇöa lesson in architecture following geometry structure.

2. **Deterministic 30 FPS Ticks**: No frame interpolation in entity logicΓÇöall state updates at fixed 30 Hz tick. `dynamic_world->tick_count` is the global simulation clock. Enables network sync and replay without floating-point drift.

3. **Deferred Operations for Netplay**: The `sDeferredObjectListInsertions` queue is a vestigial pattern from when client-side prediction had to speculatively move entities, then rollback on network updates. Suggests the codebase was heavily network-focused in Marathon 2's heyday.

4. **Garbage Collector as Gameplay Feature**: Rather than instant despawn, dead bodies and spent ammunition remain as visual clutter until quota exceeded. This is *intentional design*, not oversightΓÇöenforces map memory pressure and visual feedback of carnage. Contrast with modern games that hide corpses in 30 seconds.

5. **Vector Migration Pattern**: Comments still reference old fixed-array limits (`MAXIMUM_OBJECTS_PER_MAP`), but code now uses `ObjectList.size()`. The transition from stack-allocated to heap-allocated entities was a multi-pass modernization, not a single refactoring.

## Potential Issues

1. **Deferred Insertion List Growth**: `sDeferredObjectListInsertions` has no bounds check. In edge cases (many entities spawned in one tick), the list could grow without limit, stalling on `perform_deferred_polygon_object_list_manipulations()`.

2. **Redundant Obstruction Checks**: `line_is_obstructed()` is called once per sound source per frame; in large maps with many ambient sounds, this causes repeated polygon boundary traversals. Caching or batching could halve sound CPU cost.

3. **Silent Data Loss on Limit Exceeded**: Fixed limits like `MAXIMUM_OBJECTS_PER_MAP` are enforced at allocation, but there's no warning if exceededΓÇöcreation simply fails silently, returning `NONE`. A map designer could add one too many scenery objects and crash levels unpredictably.

4. **Garbage Eviction by Age is FIFO, Not Smart**: `turn_object_to_shit()` evicts oldest garbage when quota hit. In a scenario where monsters corpses and ammunition interleave, the wrong objects may despawn first. A weighted eviction (corpses > ammunition) would be more realistic.

5. **Polygon Object List Unlink is O(n)**: Removing an object from `polygon.first_object` chain requires traversing the list to find the previous node. With 100+ objects per polygon, removal becomes O(n). A doubly-linked list or objectΓåÆpolygon back-pointer would improve removal to O(1).
