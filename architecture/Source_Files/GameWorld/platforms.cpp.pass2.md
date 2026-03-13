# Source_Files/GameWorld/platforms.cpp - Enhanced Analysis

## Architectural Role

Platforms occupy a critical position in the dynamic geometry layer: they serve as the primary mechanism for temporal variation in walkable space, transforming static map definitions into runtime physics surfaces. This file bridges the static map definition (platform_definitions, loaded from XML) with the live world state (platforms array), and acts as the primary agent of cascading geometry updates, propagating height changes through endpoints, lines, and media subsystems. The per-frame update loop here is one of the synchronization points that keeps adjacent systems (rendering, physics, lighting) coherent.

## Key Cross-References

### Incoming (who depends on this file)
- **map.cpp**: `change_polygon_height()` receives obstruction feedback; calls back to update geometry state
- **player.cpp**: `player_touch_platform_state()` entry point for player activation; consumes key items via `try_and_subtract_player_item()`
- **monsters.cpp**: `monster_can_enter_platform()` / `monster_can_leave_platform()` used for AI pathfinding ledge checks
- **Rendering**: Platform polygon heights queried every frame during visibility/depth sorting
- **lua_script.h**: `L_Call_Platform_Activated()` hook invoked on state changes (Pfhortran scripting support)

### Outgoing (what this file depends on)
- **map.cpp**: `change_polygon_height()` (collision check + geometry update), `get_polygon_data()`, `get_map_indexes()`
- **lightsource.h**: `set_light_status()` for platform-controlled lighting
- **SoundManager.h**: `play_polygon_sound()`, ambient source updates
- **media.h**: `get_media_data()` for submersion detection
- **player.h**: Item consumption for key-locked platforms
- **platform_definitions.h** (global): Type metadata (sounds, damage, delay templates)

## Design Patterns & Rationale

**State Machine with Per-Tick Guard**: Platforms implement a disciplined state machine (active/inactive, moving/stopped, extending/contracting) with explicit per-tick limits on state changes. This pattern prevents oscillation and ensures deterministic behavior critical for networked multiplayerΓÇöa key concern in a 2000s-era codebase.

**Endpoint Owner Caching**: Rather than flood-searching for adjacent polygons/lines on every height change, `calculate_endpoint_polygon_owners()` pre-computes ownership during initialization. This trades one-time O(n) setup for O(1) per-frame geometry propagationΓÇöa pragmatic optimization for 30 FPS targets on early 2000s hardware.

**Definition + Instance + Type**: Three-layer abstraction (static_platform_data from map, platform_data runtime state, platform_definition metadata from external file) allows map authors to specify configurations while engine controls behavior. XML override (`parse_mml_platforms`) extends this without code changes.

**Kinematic, Not Physics-Based**: Platforms move via discrete height steps each tick, not continuous velocity. This simplifies collision handling (obstruction ΓåÆ stop/reverse) but sacrifices smooth accelerationΓÇöacceptable for door/elevator gameplay.

## Data Flow Through This File

```
Initialization: platform_definitions (XML) ΓåÆ new_platform() 
  ΓåÆ polygon marked _polygon_is_platform
  ΓåÆ endpoint_owner caches built (O(n) one-time)
  ΓåÆ initial state set (extended/contracted, active/inactive)

Per-frame: update_platforms() for each active platform:
  1. Compute delta_height (speed ├ù direction)
  2. Call change_polygon_height() with collision check
  3. If unobstructed:
     - Update platform.floor/ceiling_height
     - Call adjust_platform_endpoint_and_line_heights() 
     - Update media submersion flags via adjust_platform_for_media()
  4. If obstructed:
     - Reverse direction (if flag set) OR mark blocked
     - Play obstruction sound
  5. At terminal states (fully extended/contracted):
     - Deactivate if flag set
     - Activate adjacent platforms if flag set
     - Play stopping sound

State Change: set_platform_state() called by player/triggers:
  ΓåÆ Updates active flag, delay, parent_platform_index
  ΓåÆ Cascades to adjacent platforms via set_adjacent_platform_states()
  ΓåÆ Activates/deactivates associated lightsources
  ΓåÆ Calls Lua hook L_Call_Platform_Activated()
```

## Learning Notes

This file encodes idiomatic patterns of a 2000s game engine refactored from 1990s codebase:

- **Deterministic Tick-Based Simulation**: No floating-point physics; discrete height steps ensure bit-identical replay across machinesΓÇöessential before physics engines standardized.
- **Cascading Geometry Updates**: Rather than re-tessellating the entire world, smart cache invalidation (endpoint owner lists) propagates changes surgically through map.cpp's `change_polygon_height()`.
- **XML as Late-Binding Configuration**: `parse_mml_platforms()` pattern (backup originals, override definitions) shows pre-JSON era of data-driven design.
- **Lua Integration at Gameplay Layer**: Platform activation callbacks bridge scripting and core simulation without tight coupling.
- **No Continuous Movement**: Platforms teleport to new discrete heights; compare to modern engines with frame-independent continuous velocity.

## Potential Issues

- **Disabled Consistency Assertions**: Comments note disabled checks for "polygon[platform[polygon]] == platform" (line ~1200 in history) and "platform must have moving surface"ΓÇölegacy compatibility hacks for user-created Pfhorte maps.
- **Flood Status FIXME**: Lightsource recalculation assumes "Marathon 1 map lighting"ΓÇöbrittle assumption if lighting system evolves.
- **No Ceiling-Below-Floor Validation**: After `change_polygon_height()` succeeds, no sanity check that minimum_ceiling > maximum_floor; could manifest as clipping errors in subsequent frames.
- **Endpoint Owner Cache Invalidation Gap**: If adjacent geometry changes outside platform update (rare but possible), cache becomes stale until platform moves again.
