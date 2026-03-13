# Source_Files/GameWorld/placement.cpp - Enhanced Analysis

## Architectural Role

This file is the **map-specific entity ruleset loader and runtime spawning manager**. It bridges the Files subsystem (placement rules loaded from WAD) to the GameWorld subsystem (entity creation/destruction tracking). Placement rules are immutable per-map, enabling design-time tuning, deterministic replay, and consistent multiplayer synchronization. The module enforces both initial spawn logic (one-time per level) and periodic respawning (to maintain difficulty), decoupling difficulty scaling from individual entity creation code paths.

## Key Cross-References

### Incoming (who depends on this file)

**File Loading:**
- `game_wad.cpp` ΓåÆ calls `load_placement_data()` during WAD parse to initialize monster/item placement rules
- `game_wad.cpp` ΓåÆ calls `get_placement_info()` during save-game serialization to persist current placement state

**Level Initialization:**
- `marathon2.cpp` (update_world) ΓåÆ calls `place_initial_objects()` at level start
- `marathon2.cpp` (update_world) ΓåÆ calls `mark_all_monster_collections()` + `load_all_monster_sounds()` for resource preloading
- `map.cpp` ΓåÆ calls `get_random_player_starting_location_and_facing()` when spawning new players (single-player start, multiplayer respawns)

**Runtime Entity Lifecycle:**
- `monsters.cpp` (new_monster, monster_died) ΓåÆ calls `object_was_just_added/destroyed()`
- `items.cpp` (new_item, item_was_just_destroyed) ΓåÆ calls `object_was_just_added/destroyed()`
- `projectiles.cpp` (?) ΓåÆ likely calls `object_was_just_destroyed()` when projectiles create items on impact
- `marathon2.cpp` (update_world loop) ΓåÆ calls `recreate_objects()` every frame

### Outgoing (what this file depends on)

**Entity Creation / Monster Management:**
- `monsters.h::new_monster()`, `activate_monster()`, `find_closest_appropriate_target()` ΓåÆ spawn and initialize monsters
- `monsters.h::mark_monster_collections()`, `load_monster_sounds()` ΓåÆ preload monster assets (resource lifetime)
- `items.h::new_item()` ΓåÆ spawn items

**Map Geometry & Queries:**
- `map.h::polygon_data`, `get_polygon_data()`, `find_center_of_polygon()` ΓåÆ validate spawn locations
- `map.h::point_is_player_visible()`, `point_is_monster_visible()` ΓåÆ visibility culling for safe spawn points
- `map.h::find_new_object_polygon()`, `point_in_polygon()` ΓåÆ geometry navigation and wall collision
- `map.h::get_player_starting_location_and_facing()` ΓåÆ retrieve predefined player spawn points from map

**Global Mutable State:**
- `dynamic_world->current_monster_count[]`, `current_item_count[]` ΓåÆ live entity counters (updated by add/destroy callbacks)
- `dynamic_world->random_monsters_left[]`, `random_items_left[]` ΓåÆ remaining random spawn budget per type
- `dynamic_world->tick_count` ΓåÆ frame timing (respawn interval gating)
- `GET_GAME_OPTIONS()` ΓåÆ `_monsters_replenish` flag (enables/disables respawning)
- `film_profile.initial_monster_fix` ΓåÆ compatibility override for version-specific behavior

## Design Patterns & Rationale

**Single Combined Array with Pointer Slicing**  
`object_placement_info[2*MAXIMUM_OBJECT_TYPES]` is split into monsters (`+MAXIMUM_OBJECT_TYPES`) and items (`+0`), with pointers `monster_placement_info` and `item_placement_info` indexing each half. This is a space-efficient savings for save-game serialization (single contiguous WAD block) at the cost of manual pointer arithmetic. Trade-off: simpler I/O but fragile if array layout changes.

**Immutable Ruleset ΓåÆ Deterministic Difficulty Scaling**  
Difficulty is baked into placement rules (initial_count, min_count, max_count, random_chance) loaded per-map, not computed at runtime. This enables:
- Level designers to tune (not programmers)
- Multiplayer sync (same rules ΓåÆ same entity distribution across clients)
- Deterministic replay (film playback reproduces identical spawn events)

**Dual Spawn Strategies**  
`is_initial_drop` flag changes behavior:
- **Initial**: Pick from predefined map locations (`pick_random_initial_location_of_type`), respects visibility culling
- **Respawn**: Pick random polygon center (`choose_invisible_random_point`), more permissive but retry-limited

Rationale: Initial spawn uses designer intent; respawning uses mechanical safeguards (polygon type checks, collision avoidance).

**Lazy Respawn via Minimum-Count Maintenance**  
Rather than spawn monsters on a timer, the system checks min/max/random constraints at periodic intervals. If counts dip below minimum (e.g., player killed them), one is respawned immediately. This prevents player from trivializing difficulty via massacre, decouples respawn from elapsed time.

**Polymorphic Iteration (_recreate_objects)**  
A single function handles both monsters and items via parameters (object_type, max_object_types, placement_info pointers, count arrays). Avoids code duplication but obscures the fact that monsters and items have different ruleset columns (e.g., marine is skipped for monsters).

**Wall-Aware Facing Direction (pick_random_facing)**  
Newly spawned monsters get a facing angle that "points outward" (into adjacent polygons, away from walls). Tests 4 cardinal directions; falls back to random if stuck. Purpose: monsters don't spawn facing walls and immediately pathfind away; immersion + performance.

## Data Flow Through This File

**Load Phase (Initialization):**
```
WAD file stream (binary bytes)
  Γåô
load_placement_data(uint8 *_monsters, uint8 *_items)
  Γåô unpack_object_frequency_definition()
object_placement_info[] (struct object_frequency_definition[2*MAXIMUM])
  Γåô pointer slicing
monster_placement_info, item_placement_info (ready for place/recreate)
```

**Initial Spawn Phase (Level Start):**
```
place_initial_objects()
  ΓåÆ For each monster type (1..N):
      if (initial_count > 0 && conditions met)
        add_objects(_object_is_monster, type, initial_count, is_initial_drop=true)
  ΓåÆ For each item type (0..N):
      if (initial_count > 0)
        add_objects(_object_is_item, type, initial_count, is_initial_drop=true)
  ΓåÆ Sync dynamic_world counters and random budgets
  
add_objects()
  ΓåÆ For each instance to create:
      Decide location strategy:
        - initial_drop=true ΓåÆ pick_random_initial_location_of_type() [predefined]
        - initial_drop=false ΓåÆ choose_invisible_random_point() [random]
      Pick facing direction ΓåÆ pick_random_facing()
      Validate polygon ΓåÆ polygon_is_valid_for_object_drop()
      Create entity ΓåÆ new_monster() or new_item()
      For non-initial: activate_monster(), assign target
```

**Runtime Respawn Phase (Per-Frame Loop):**
```
recreate_objects() [called from update_world ~30 FPS]
  Every ~15 ticks (spawn interval):
    _recreate_objects(object_class, ..., placement_info, object_counts[], random_counts[])
      ΓåÆ For each type:
          diff = min_count - current_count
          if (diff > 0): add_objects(..., is_initial_drop=false)
          if (random & conditions met): add_objects(..., 1, is_initial_drop=false)
            ΓåÆ decrement random_counts[type]
```

**Lifecycle Tracking (Entity Callbacks):**
```
object_was_just_added(class, type)
  ΓåÆ dynamic_world->current_count[type]++

object_was_just_destroyed(class, type)
  ΓåÆ dynamic_world->current_count[type]--
  ΓåÆ diff = min_count - current_count
  ΓåÆ if (diff > 0): add_objects(..., 1, is_initial_drop=false)
```

**Key State Transitions:**
- Marine (index 0) forced to zero (never spawns as regular entity, only as player)
- Random budget decrements only if `random_count != NONE` (NONE = infinite spawning)
- Respawn respects maximum_count (never over-spawn)

## Learning Notes

**This is a Mid-90s Map Design Philosophy**

Place.cpp encodes a ruleset system common to Marathon, Doom, QuakeΓÇödesigners define entity distributions as static lookup tables, not procedural generation or distance-LOD systems. Immutable per-map rules enabled:
- **Authorial control**: levels balanced by hand tuning, not emergent difficulty
- **Deterministic replay**: film playback and network sync worked because spawning was pre-computed
- **Offline design**: artists could test maps before shipping without live server infrastructure

**Modern Engines Diverge:**
- **LOD-based spawning**: Unreal/Unity spawn/despawn based on camera distance
- **Trigger-driven spawning**: Designer places explicit spawn volumes, not automatic rules
- **Dynamic difficulty**: Adapts to player performance in real-time
- **Proc-gen**: Generates encounters algorithmically

**Hardcoded Assumptions:**
- Monster index 0 is always the player avatar (marine); never spawns in-world
- Player starting locations are predefined on the map (not dynamic)
- Item placement respects minimum counts (won't run out of healing/ammo)
- Monsters and items use separate lookup tables (could have been unified)

**Idiosyncratic Design Choices:**
- **Visibility culling on initial spawn**: Rejects placements where player is looking (immersion), but tight maps might fail silently
- **Retry-limited random location**: 10 attempts; if all fail, add_objects silently gives up (no warning)
- **Facing direction calculation**: Only tests 4 cardinal directions; assumes cardinal-aligned geometry (Marathon's maps were axis-aligned)
- **Film profile compat flag**: `initial_monster_fix` suggests legacy behavior changed; hints at version migration issues

## Potential Issues

1. **Silent Failure on Random Spawn Exhaustion**  
   `choose_invisible_random_point()` retries only 10 times. If map is packed, objects may not spawn, and no error is raised. Players might see fewer enemies than expected without warning.

2. **Marine Hardcoding Fragility**  
   Monster index 0 is forced to zero in three places (`load_placement_data`, `place_initial_objects`, `_recreate_objects`). If monster definitions change, this assumption breaks silently. No enum constant guards the assumption.

3. **Disabled Network-Only Item Code (#if 0)**  
   Block at lines ~80-130 (in full file) filters network-only items in single-player. Code is commented outΓÇösuggests past bug or unfinished feature. If re-enabled, could cause spawn desync.

4. **Pointer Arithmetic Risk**  
   `monster_placement_info = object_placement_info + MAXIMUM_OBJECT_TYPES` is correct but fragile. If array is reallocated or resized incorrectly, pointers dangle. No bounds checking.

5. **Determinism Assumptions (Multiplayer)**  
   Respawning uses `global_random()` (presumably seeded per-match). If seed diverges or RNG differs across clients, spawn timing/locations diverge ΓåÆ desync. No explicit synchronization mechanism visible here.

6. **Visibility Check Scope**  
   `polygon_is_valid_for_object_drop(..., is_random_location)` behavior differs for predefined vs. random locations, but the validation criteria are not symmetric. Edge case: predefined location fails visibility check ΓåÆ add_objects may silently skip.
