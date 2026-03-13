# Source_Files/GameWorld/projectiles.cpp - Enhanced Analysis

## Architectural Role

Projectiles.cpp forms a cornerstone of the GameWorld simulation alongside monsters.cpp, items.cpp, and player.cpp. It bridges **weapon firing logic** (called by weapons.cpp) with the **main game loop** (move_projectiles invoked from marathon2.cpp's update_world). Projectiles act as the kinetic intermediary between combat intent and damage resolution, making this module critical to gameplay feel and balance across difficulty levels.

## Key Cross-References

### Incoming (who depends on this file)
- **weapons.cpp**: Calls `new_projectile()` when player/NPC weapons fire; passes owner, type, and intended target
- **marathon2.cpp** (main loop): Calls `move_projectiles()` once per 30 FPS tick to advance all active projectiles
- **Lua scripts** (lua_script.h): Can query/manipulate projectiles; receives `L_Call_Projectile_Created` and `L_Call_Projectile_Detonated` callbacks
- **Weapon code** (implicit): Calls `preflight_projectile()` to validate firing legality before calling new_projectile

### Outgoing (what this file depends on)
- **monsters.cpp**: `damage_monsters_in_radius()`, `damage_monster()`, `get_monster_data()`, `get_monster_dimensions()`, `get_monster_impact_effect()`, `get_monster_melee_impact_effect()` ΓÇö core damage/targeting pathway
- **map.cpp**: `translate_projectile()` collision detection, `new_map_object3d()`, `remove_map_object()`, `animate_object()`, `get_polygon_data()`, `find_line_*` ΓÇö spatial queries and object lifecycle
- **effects.h/cpp**: `new_effect()` for detonation visuals
- **media.h/cpp**: `get_media_data()`, `get_media_detonation_effect()` ΓÇö media boundary and splash effects
- **items.h/cpp**: `new_item()` when projectiles become items on detonation
- **player.h/cpp**: `try_and_add_player_item()`, `monster_index_to_player_index()` ΓÇö item handoff to players
- **SoundManager.h**: `play_object_sound()` for projectile sounds
- **projectile_definitions.h**: Global projectile_definitions array, difficulty scaling lookup

## Design Patterns & Rationale

**Object Pool + Slot Marking**: Projectiles live in a pre-allocated `projectile_data` array; `SLOT_IS_USED/SLOT_IS_FREE` macros manage allocation. This avoids dynamic allocation in the 30 FPS hot path. Free slots are recycled by linear scan in `new_projectile()`.

**Collision Delegation to `translate_projectile()`**: All collision tests (landscape, ceiling, floor, media, monsters, scenery) are outsourced to a single function that returns a flags bitmask (`_projectile_hit_*`). This isolates spatial complexity and makes the main `move_projectiles()` loop readable.

**Definition-Driven Behavior**: Each projectile type stores configuration in `projectile_definition` (speed, gravity flags, area-of-effect, damage, detonation effects). The same projectile instance can have different behavior based on its definition, enabling:
- Copy-protection via `alien_projectile_override` / `human_projectile_override`
- Difficulty-scaled alien speeds via switch on `dynamic_world->game_information.difficulty_level`
- Feature flags (guided, bouncing, penetrating, melee, bleeding, persistent) controlling behavior conditionally

**Media Boundary Penetration Evolution**: Comments (Feb 2000 Loren Petrich) show this feature evolved through iterations. The "penetrates media boundary" flag now means "splash on media surface but continue moving"ΓÇöachieved by:
1. Setting `will_go_through = true` on media hit
2. Manually nudging projectile position (z++ or z--) to escape the media layer
3. Allowing continued movement for additional detonations

## Data Flow Through This File

```
CREATION:
  weapons.cpp ΓåÆ new_projectile(origin, type, vector, owner, target, damage_scale)
    ΓåÆ adjust_projectile_type() [apply copy-protection overrides]
    ΓåÆ new_map_object3d() [create renderable object]
    ΓåÆ initialize projectile_data slot (owner, target, flags, gravity, elevation, distance)
    ΓåÆ L_Call_Projectile_Created() [Lua callback]

MOVEMENT (per tick in move_projectiles):
  for each active projectile:
    ΓåÆ animate_object() [update sprite frame]
    ΓåÆ apply gravity (half/full/double based on flags)
    ΓåÆ update_guided_projectile() [if guided + valid target, adjust facing/elevation every 2 ticks]
    ΓåÆ translate_projectile(old_location ΓåÆ new_location)
      ΓåÉ returns collision flags + obstruction_index
    ΓåÆ IF HIT DETECTED:
        Γö£ΓöÇ IF rebound-eligible: bounce gravity
        ΓööΓöÇ ELSE:
            Γö£ΓöÇ damage_scenery() if hit scenery
            Γö£ΓöÇ IF not yet damaged:
            Γöé  Γö£ΓöÇ damage_monsters_in_radius() / damage_monster() / becomes_item_on_detonation
            Γöé  ΓööΓöÇ SET_PROJECTILE_DAMAGE_STATUS
            Γö£ΓöÇ IF media hit + penetrates_media_boundary:
            Γöé  ΓööΓöÇ splash + nudge position + continue
            ΓööΓöÇ new_effect(detonation_effect, ...)
            ΓööΓöÇ remove_projectile() [unless persistent flag]
    ΓåÆ ELSE: track contrail, check max range, continue

REMOVAL:
  remove_projectile() ΓåÆ remove_map_object() ΓåÆ L_Invalidate_Projectile()
```

## Learning Notes

**Difficulty Scaling for Alien Weapons**: Alien projectile speeds are scaled per difficulty (wuss: -12.5%, easy: -6.25%, major damage: +12.5%, carnage: +25%). This balances combat without changing behavior flagsΓÇöelegant for rebalancing without data changes.

**Preflight vs. Actual Translation**: `preflight_projectile()` does a dry-run collision test (with `preflight=true` flag) to validate weapon firing before actually creating the projectile. This is idiomatic for MarathonΓÇöpreventing "shots in solid geometry" without runtime errors.

**Guided Projectiles with Target Locking**: Guided projectiles update targeting **every other tick** (`dynamic_world->tick_count&1`) to reduce CPU cost. Targeting is broken if the target becomes invisible (except on total carnage), allowing skill-based target dodging.

**IntersectedObjects: Growable List Optimization**: The static `vector<short> IntersectedObjects` accumulates monster indices during collision checks in a single polygon. This avoids repeated allocations during `translate_projectile()` and allows easy clearing between frames.

**Persistent Projectiles**: Some projectiles (marked `_persistent`) survive first impact; they only despawn after causing damage + hitting something solid. This enables "bouncing grenades" and "ricocheting needles" mechanics.

## Potential Issues

1. **Unbounded IntersectedObjects Growth**: The `IntersectedObjects` vector is never explicitly pruned; in extreme polygon-stacking scenarios, it could accumulate indices and never shrink, wasting memory.

2. **Media Boundary Penetration Complexity**: The PMB logic (lines ~700ΓÇô750) manually tweaks `new_location.z` based on elevation direction. Edge cases could occur if projectile elevation is exactly 0 or HALF_CIRCLE; the current code uses empty if-blocks for these, risking missed behavior.

3. **Copy-Protection Overrides at Creation Time**: `alien_projectile_override` and `human_projectile_override` are checked once in `new_projectile()`. If these globals are modified mid-game, existing projectiles won't be affectedΓÇöonly newly-fired ones. This could cause UI/balance inconsistencies during replays or mod testing.

4. **Collision Detection Correctness**: The `translate_projectile()` function is ~1000 LOC (truncated in excerpt) and handles many edge cases (transparent lines, Pfhorte workarounds, media transitions). Subtle bugs in path calculation or flag propagation could cause projectiles to phase through scenery or register false impacts.
