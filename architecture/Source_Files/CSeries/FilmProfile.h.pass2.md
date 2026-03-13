# Source_Files/CSeries/FilmProfile.h - Enhanced Analysis

## Architectural Role

FilmProfile implements a **runtime film compatibility layer** enabling Aleph One to replay films recorded in different game versions (Marathon 1/2/Infinity, Aleph One 1.0ΓÇô1.7) by conditionally applying version-specific behavior at runtime. Rather than maintaining separate engine builds, it centralizes behavior quirks and bug fixes as boolean toggles evaluated throughout GameWorld physics, damage calculations, spawning logic, and rendering to ensure deterministic film playback. The struct acts as a **version-aware configuration card** loaded during film initialization via `load_film_profile()`.

## Key Cross-References

### Incoming (who depends on this file)

- **Files/Videos subsystem** ΓÇô calls `load_film_profile()` when initializing film playback (inferred from film loader usage)
- **Interface/Shell** ΓÇô invoked during level transitions or film loading; comment explicitly mentions `interface.cpp` must be updated when profiles are added
- **GameWorld subsystem** ΓÇô checks individual `film_profile.*` flags in:
  - `physics.cpp` ΓÇô `long_distance_physics`, `m1_low_gravity_projectiles` 
  - `items.cpp` ΓÇô `swipe_nearby_items_fix`, `animate_items`, `m1_bce_pickup`
  - `monsters.cpp` ΓÇô `initial_monster_fix`, likely pathfinding/spawning logic
  - `platforms.cpp` ΓÇô `fix_sliding_on_platforms`, `m1_platform_flood`
  - `weapons.cpp` / `projectiles.cpp` ΓÇô `use_vertical_kick_threshold`, `m1_landscape_effects`
  - `damage/collision logic` ΓÇô `damage_aggressor_last_in_tag`, `infinity_tag_fix`, `adjacent_polygons_always_intersect`
- **RenderMain subsystem** ΓÇô `line_is_obstructed_fix`, `a1_smg` / `infinity_smg` physics variants
- **Lua subsystem** ΓÇô `lua_increments_rng`, `lua_monster_killed_trigger_fix`, `early_object_initialization` (timing of Lua callbacks)

### Outgoing (what this file depends on)

- No dependencies on other files (pure data structure definition)
- Global `film_profile` is **read-only** throughout the engine (no external writes)
- `FilmProfileType` enum references correspond 1:1 to profile definitions in implementation file (likely `interface.cpp` or `preferences.cpp`)

## Design Patterns & Rationale

**Strategy Pattern (Runtime):** Each `FilmProfileType` selects a different behavioral strategy. Rather than subclassing or function pointers, simple boolean flags allow inline conditional evaluation with zero indirection cost.

**Feature Flag Design:** 45+ individual toggles instead of monolithic "version mode" allows granular control: a single fix can apply across multiple versions (e.g., `better_terminal_word_wrap` fixes "rare infinity films" only).

**Documentation as Code:** Every flag includes an inline comment naming the version, the behavioral change, and (often) the author. This creates a **live changelog embedded in the engine**, invaluable for debugging cross-version determinism issues.

**No Inheritance / Polymorphism:** Pragmatic choice for an older codebase (BungieΓåÆcommunity project). Avoids vtable overhead; enables zero-cost checks via branch prediction and inlining.

## Data Flow Through This File

1. **Film Loading:** Replay system determines film's source version ΓåÆ calls `load_film_profile(FilmProfileType)` 
2. **Profile Activation:** `load_film_profile()` (impl in separate `.cpp`) populates global `extern FilmProfile film_profile`
3. **Runtime Checking:** Throughout GameWorld loop (30 FPS tick), subsystems check `if (film_profile.flag_name) { use_v1_behavior() } else { use_current_behavior() }`
4. **Deterministic Replay:** Conditional paths ensure physics, damage, AI pathfinding, item spawning, and platform behavior match original recording exactly

Example: `film_profile.long_distance_physics` ΓåÆ affects `arctangent()` and `distance2d()` calls in world.cpp, ensuring distant projectiles behave identically to their source engine.

## Learning Notes

- **Historical Artifact of Version Fragmentation:** Marathon 1/2/Infinity were developed by different teams with diverging physics models. This file is proof that compatibility requires emulating bugs, not just features.
- **Determinism as Core Value:** Unlike modern engines (which might netcode-sync or server-authoritative), Aleph One enforces local determinismΓÇöthe same recording must produce identical output on any machine.
- **Incremental Accumulation:** Each Aleph One release (1.0 ΓåÆ 1.7) added new fixes; older profiles inherit earlier fixes. This forms a **version hierarchy**, not independent profiles.
- **Idiomatic to This Era:** Pre-2010s engines often used massive feature flag tables like this. Modern engines prefer data-driven "behavior packs" (JSON/YAML configs) or hot-reloading script systems.

## Potential Issues

- **Profile Completeness:** Adding a new version-specific behavior requires both a new flag AND updating `load_film_profile()` implementation + interface.cpp. Comment warns of this but no mechanism enforces it (lint check or enum reflection would help).
- **Flag Scope Creep:** 45+ flags suggest the struct may have crossed the "small focused concern" threshold; difficult to reason about interactions between flags (e.g., does `infinity_tag_fix` interact with `damage_aggressor_last_in_tag`?).
- **No Version Validation:** No checksum or version ID stored in `FilmProfile` itself; relies entirely on caller to pass correct enum value. Silent misbehavior if caller guesses wrong version.
