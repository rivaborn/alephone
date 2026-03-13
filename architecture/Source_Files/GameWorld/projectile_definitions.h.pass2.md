# Source_Files/GameWorld/projectile_definitions.h - Enhanced Analysis

## Architectural Role

This file is the **single source of truth for projectile behavior tuning** in the Aleph One game engine. It sits at the intersection of physics (GameWorld), audio (SoundManager), visual effects (Effects), and world geometry (Map/Media), acting as a data hub that decouples projectile specifications from the simulation code that consumes them. The const-mutable dual-array pattern (`original_projectile_definitions[]` + `projectile_definitions[]`) enables both immutable defaults and runtime customization, supporting mod systems, map-embedded overrides, and deterministic replay.

## Key Cross-References

### Incoming (who depends on this file)
- **projectiles.cpp**: Reads `projectile_definitions[]` on spawn (speed, damage, radius, effects, flags)
- **game_wad.cpp**: Calls `unpack_projectile_definition()` / `pack_projectile_definition()` to persist/restore custom definitions in save games and map files
- **weapons.cpp**: Queries projectile properties when firing (speed, area-of-effect, guided flag)
- **marathon2.cpp** (main loop): Indirectly accesses via projectile simulationΓÇödamage calculations, detonation checks
- **render.cpp**: Uses collection/shape indices for visual rendering of in-flight projectiles
- **SoundManager**: Plays flyby/rebound sounds indexed by `flyby_sound` and `rebound_sound` fields

### Outgoing (what this file depends on)
- **effects.h**: Detonation effect types (`_effect_rocket_explosion`, `_effect_grenade_explosion`, etc.)
- **map.h**: `damage_definition` struct, world distance types (`WORLD_ONE`), spatial scale constants
- **media.h**: Media detonation effects (`_small_media_detonation_effect`, `_medium_media_detonation_effect`)
- **projectiles.h**: `NUMBER_OF_PROJECTILE_TYPES` constant; index mapping for projectile enums
- **SoundManagerEnums.h**: Sound event IDs (`_snd_rocket_flyby`) and frequency modulation constants

## Design Patterns & Rationale

**Data-Driven Configuration**  
All 22+ behavior flags + 10+ numeric parameters per projectile type are encoded declaratively rather than in code branches. This enables:
- **Easy modding**: Adjust damage/speed without recompilation
- **Granular control**: Turnable damage scaling per difficulty (via `_alien_damage` damage type)
- **Feature evolution**: New flags added without restructuring (Feb 2000 SMG liquid penetration example)

**Bitflag Composition**  
Flags like `_guided|_can_toggle_control_panels|_melee_projectile` combine independently, avoiding explosion of projectile type variants. The 32-bit `uint32 flags` field currently uses ~22 bits; extensible for future features.

**Const-Mutable Duality**  
`original_projectile_definitions[]` is `const` (compile-time immutable) while `projectile_definitions[]` is `static` (runtime mutable). This pattern:
- Protects defaults from accidental mutation
- Allows deterministic replay (restore from const master if needed)
- Supports per-game customization via deserialization
- Enables "reset to defaults" by memcpy from const array

**Serialization Bridge**  
`pack_projectile_definition()` / `unpack_projectile_definition()` decouple in-memory representation from disk/network format, enabling:
- Map files to carry embedded projectile overrides (campaign-specific balance)
- Network sync of custom definitions in multiplayer
- Save games to preserve tuning changes

**Media Type Promotion**  
The `media_projectile_promotion` field enables **projectile mutation** (e.g., SMG bullet in water becomes a different projectile type). This is a form of state-machine dispatch at the data level.

## Data Flow Through This File

```
Compile-time:
  original_projectile_definitions[40]  ΓåÉ Hard-coded definitions
                                           (all 40 projectile types)

Load-time (map/game load):
  map file / save game
       Γåô
  unpack_projectile_definition()  ΓåÉ Deserialize custom definitions
       Γåô
  projectile_definitions[40]  ΓåÉ Runtime mutable copy

Runtime (each frame):
  projectiles.cpp::new_projectile()
       Γåô
  projectile_definitions[type_index]  ΓåÉ Read speed, damage, flags
       Γåô
  Spawn projectile with behavior derived from definition
       Γåô
  Per-frame: check flags (_affected_by_gravity, _guided, etc.)
       Γåô
  Detonation: read _effect_rocket_explosion, play sounds

Save-time:
  projectile_definitions[40]  ΓåÉ Modified at runtime
       Γåô
  pack_projectile_definition()  ΓåÉ Serialize to disk
       Γåô
  save game
```

## Learning Notes

**Era-appropriate design**: This 1994-era codebase predates modern ECS and data-driven engines, yet uses a clean declarative configuration pattern. The `projectile_definition` struct is essentially a **typed data record**ΓÇöexactly what modern JSON/YAML config would do, implemented in C structs.

**Extensibility in practice**: The Feb 2000 addition of `_penetrates_media_boundary` for SMG bullets shows how the engine evolved post-ship. Rather than creating a new projectile type, they added a flag and updated the flag dispatch logic in `projectiles.cpp`. The bitflag system absorbed this gracefully.

**Difficulty scaling**: The `_alien_damage` damage type in alien projectile definitions shows how balance tuning is baked into dataΓÇömonsters can deal reduced damage on lower difficulties via damage type lookup, not code conditionals.

**Contrails as side effects**: The contrail system (`contrail_effect`, `ticks_between_contrails`, `maximum_contrails`) is elegantly modeled as a projectile property rather than hard-coded smoke trailsΓÇövisual effects are data, not logic.

## Potential Issues

- **Flag space exhaustion**: 32-bit `flags` field has ~10 bits remaining. If 10+ new behaviors are needed, refactoring to a larger field or separate struct is necessary.
- **Single media promotion**: `media_projectile_promotion` can only specify one alternate type. If a projectile needs different behavior in water vs. lava vs. goo, this doesn't scale. Would need a promotion table per projectile type.
- **Const-mutable footgun**: Developers unfamiliar with the pattern might accidentally read from mutable `projectile_definitions[]` expecting determinism, or write to const `original_projectile_definitions[]` via const_cast. Clear naming (e.g., `default_projectile_definitions[]` vs. `active_projectile_definitions[]`) might reduce confusion.

---

**Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>**
