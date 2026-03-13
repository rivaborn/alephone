# Source_Files/RenderOther/motion_sensor.cpp - Enhanced Analysis

## Architectural Role

The motion sensor bridges **GameWorld simulation** and **HUD rendering**, providing a filtered, real-time tactical overlay of nearby entities. It sits in the `RenderOther/` subsystem, acts as a periodic **world observer** (not a participant), and demonstrates how Aleph One decouples simulation from view by maintaining a separate entity tracking cache with dual-frequency updates and smooth fade animation. The file supports three rendering backends (SW bitmap, OpenGL immediate, Lua scripting), each with distinct visual semantics but unified data model.

## Key Cross-References

### Incoming (who depends on this file)
- **GameWorld/marathon2.cpp** ΓåÆ calls `motion_sensor_scan()` each game tick (30 Hz), driving periodic world polling
- **RenderOther/HUDRenderer_SW/OGL/Lua.cpp** ΓåÆ call `render_motion_sensor()` and `motion_sensor_has_changed()` to render/update display
- **Misc/interface.cpp** ΓåÆ calls `initialize_motion_sensor()` during engine startup to pass shape descriptors and allocate backing storage
- **GameWorld/map.cpp** (level load) ΓåÆ calls `reset_motion_sensor()` per level to clear state and restore virgin bitmap
- **Global config tables** ΓåÆ `MonsterDisplays[]` read by `get_motion_sensor_entity_shape()` to determine entity display type per monster index

### Outgoing (what this file depends on)
- **GameWorld/map.h** ΓåÆ `get_object_data()`, `get_monster_data()`, `get_player_data()` (entity state queries)
- **GameWorld/world.h** ΓåÆ `guess_distance2d()`, `transform_point2d()` (2D math for range checks and coordinate transforms)
- **RenderMain/shape_descriptors.h** ΓåÆ shape descriptor lookups and bitmap accessor functions
- **Network/network_games.h** ΓåÆ `get_network_compass_state()` (multiplayer team indicators in corners)
- **RenderOther/screen_drawing.h** ΓåÆ bitmap copy/clear operations for SW backend (mount restoration)
- **Per-object transfer mode checks** ΓåÆ `_xfer_invisibility`, `_xfer_subtle_invisibility` (entity visibility determination)

## Design Patterns & Rationale

**Dual-Frequency Update Model**: Rescan (15 Hz) discovers new entities within range; Update (6 Hz) animates positions and fade. This separates **discovery from rendering** and amortizes world-iteration cost across framesΓÇöthe 1990s engine could not afford to scan all monsters every frame at 30 Hz.

**Position History as Fade Animation**: Rather than instant removal, entities are marked "being removed" and their `previous_points[6]` history animates across frames. Creates visual **signal decay** mimicking radar fade and hides LOD transitions when entities pop off-screen.

**Entity Pool with Removal Delay**: Fixed 12-entity array avoids dynamic allocation but requires graceful removal. The `remove_delay` counter (0ΓÇô5) ensures a tracked entity's blip fades across 6 update cycles before the slot is freedΓÇöprevents flicker when entities die.

**Precomputed Circular Region**: `precalculate_sensor_region(side_length)` computes `x0/x1` clip boundaries per scanline once at init, then `clipped_transparent_sprite_copy()` uses these to clip blips to circular boundary **without per-pixel distance checks during rendering**.

**Backend-Agnostic Data, Rendering-Specific Display**: `entity_data` (position history, visibility) is backend-agnostic; three overloads of `render_motion_sensor()` handle SW (bitmap ops), OGL (shape draw), and Lua (custom script). Lua variant is notably minimalΓÇöjust calls `draw_all_entity_blips()`, delegating visuals to script.

**Dirty-Flag Optimization for Software Renderer**: `motion_sensor_changed` flag skips redraw on static display (SW backend only). OGL/Lua backends redraw every frame since GPU/script overhead is acceptable; SW bitmap blitting is expensive, so conditional redraws matter.

**Monster Type ΓåÆ Display Mapping Table**: `MonsterDisplays[NUMBER_OF_MONSTER_TYPES]` maps monster index to `MType_Friend`/`MType_Alien`/`MType_Enemy`. MML-configurable, allowing per-scenario display customization without recompilationΓÇöseen in comments: "since it's now read off of a table, it can easily be changed without rebuilding the app" (Mar 23, 2000).

## Data Flow Through This File

```
INIT: initialize_motion_sensor()
  Γö£ΓöÇ Store shape descriptors (mount, aliens, friends, enemies, compass)
  Γö£ΓöÇ Allocate: entity_data[12], region_data[side_length]
  ΓööΓöÇ Precompute circular clip boundaries

LEVEL_START: reset_motion_sensor(player_index)
  Γö£ΓöÇ Restore virgin mount bitmap (clean slate)
  Γö£ΓöÇ Clear all entity slots (flags = 0)
  ΓööΓöÇ Reset compass state

PER_TICK: motion_sensor_scan()
  Γö£ΓöÇ If (--ticks_since_last_rescan < 0):
  Γöé  Γö£ΓöÇ Iterate all monsters in world
  Γöé  Γö£ΓöÇ Distance-check & visibility-check each
  Γöé  ΓööΓöÇ find_or_add_motion_sensor_entity() ΓåÆ populate tracking array
  ΓööΓöÇ If (--ticks_since_last_update < 0):
     ΓööΓöÇ erase_all_entity_blips()
        Γö£ΓöÇ Check each tracked entity still in range/alive
        Γö£ΓöÇ Shift position history (memmove [1..5] ΓåÆ [0..4], clear [5])
        Γö£ΓöÇ Compute new 3DΓåÆ2D world-to-sensor projection (if active)
        ΓööΓöÇ Increment remove_delay for entities marked "being removed"

PER_FRAME: render_motion_sensor()
  Γö£ΓöÇ [SW] Restore virgin mount, call draw_all_entity_blips()
  Γö£ΓöÇ [OGL] Draw virgin mount shape, compass, draw_all_entity_blips()
  ΓööΓöÇ [Lua] Call draw_all_entity_blips() only

draw_all_entity_blips()
  ΓööΓöÇ For intensity=5 down to 0 (oldest ΓåÆ newest):
     ΓööΓöÇ For each tracked entity:
        ΓööΓöÇ If visible_flags[intensity]: draw_entity_blip(pos[intensity], shape+intensity)
        [Layering: fainter blips drawn first so newer blips overlay them]
```

## Learning Notes

**Decoupled Observation Pattern**: Motion sensor never modifies game stateΓÇöit only reads and caches. This exemplifies the **Observer pattern** common in 1990s game engines before modern ECS architecture. Useful for HUD elements that mirror world state without entanglement.

**Coordinate Transformation**: `transform_point2d((world_point2d *)&entity->previous_points[0], owner_object->location, facing)` rotates entity positions into player-relative frame. This is standard **egocentric view** transformation (player at origin, forward-facing up). Modern engines do this in shaders; here it's per-entity per-update in CPU code.

**Magnetic Environment Flicker**: `(!(static_world->environment_flags&_environment_magnetic) || !((dynamic_world->tick_count+4*monster->object_index)&FLICKER_FREQUENCY))` implements a **deterministic per-entity flicker** in magnetic rooms. The `4*monster->object_index` spreads flicker timing across monsters so they don't all blink in syncΓÇöa visual detail from 1994 Marathon.

**Shape Descriptor Polymorphism**: `get_motion_sensor_entity_shape()` selects shape based on monster type, player team, game mode. This avoids per-pixel color-coding by using **visually distinct sprite shapes** (alien squiggle vs. friendly circle). Data-driven via `MonsterDisplays[]`.

**Idiomatic 90s Data Structure**: `entity_data` uses manual bitfield flags (`SLOT_IS_USED_BIT`) and hand-rolled macros (`MARK_SLOT_AS_FREE()`). Modern code would use bool fields; but this packed struct saves memory on limited RAM (e.g., Mac PowerPC from 1994).

**Time-Slicing Heuristic**: Separating rescan (15 Hz) from update (6 Hz) is a **manual frame-time budgeting** strategy. Modern engines use profiling and dynamic budgeting; Aleph One uses fixed-frequency heuristics tuned through playtesting.

## Potential Issues

**Silent Capacity Exhaustion**: If >12 entities within range, `find_or_add_motion_sensor_entity()` returns `NONE` without adding them. Player sees incomplete radar. Compounded by **no visual indication** of which entities are tracked vs. off-display. See comment in first-pass: "Skips adding if all 12 slots full."

**Monster Slot Reuse Bug**: Comment in `erase_all_entity_blips()`: "sometimes we might start tracking a new monster which has been placed in our old monster's slot, but we eat that chance without remorse." If a 20-entity dies and a new 47-entity spawns at index 20, the motion sensor may display an icon for entity 47 but track entity 20's old position briefly. Low-impact but lurking.

**Transfer Mode Hardcoding**: Visibility check is `object->transfer_mode!=_xfer_invisibility && object->transfer_mode!=_xfer_subtle_invisibility`. If a new invisibility mode is added (e.g., `_xfer_phase_cloak`), this code must be updated. No extensibility point via table lookup.

**No Memory Cleanup**: `initialize_motion_sensor()` allocates `entities` and `sensor_region` with `new[]` but no destructor or cleanup function defined. If the engine reinitializes the motion sensor mid-game (unlikely but possible), memory leaks.

**Precision Loss in Position Comparison**: `entity->last_location` vs. `object->location` comparison uses direct equality on fixed-point or float coordinates. Very slow movements or floating-point rounding could cause entities to appear stationary on radar when they're actually moving 1 unit/frame.

**Magnetic Flicker Determinism**: The flicker calculation `(dynamic_world->tick_count+4*monster->object_index)&FLICKER_FREQUENCY` depends on global tick counter and object index, making it deterministic and repeatable. However, if monster indices are reused dynamically, flicker patterns could appear jumpy. Mitigated by 1994-era low object churn, but worth noting.
