# Source_Files/CSeries/FilmProfile.cpp - Enhanced Analysis

## Architectural Role

FilmProfile.cpp implements **forward/backward compatibility** for the Aleph One engine by maintaining a registry of behavioral configurations tied to specific game/engine versions. It acts as a **compatibility shim** between replay/film loading and the rest of the engine: when a replay from Marathon 2, Marathon Infinity, or a specific Aleph One version is loaded, `load_film_profile()` switches the global `film_profile` to match that version's quirks and fixes. Every affected subsystem (physics, AI, rendering, Lua, UI) queries this global flag set at decision points to replicate legacy behavior. This design preserves replay determinism and allows old saves/films to play identically on newer engine versions.

## Key Cross-References

### Incoming (who calls into this file)
- **Film/Replay loading** ΓÇö The `film_profile` must be switched when a replay is loaded (likely called from `Videos/Movie.cpp` or film header parsing)
- **Recorded input playback** ΓÇö When replaying stored actor input, the engine must use the recorded version's profile to guarantee deterministic replay
- **Game initialization** ΓÇö Default game setup likely calls `load_film_profile(FILM_PROFILE_DEFAULT)` to activate `alephone1_11`

### Outgoing (what this file affects)
The static profiles here are queried by multiple subsystems via the global `film_profile`:

- **Physics** (`long_distance_physics`, `m1_low_gravity_projectiles`) ΓÇö `Source_Files/GameWorld/physics.cpp` 
- **Monster AI** (`initial_monster_fix`, `validate_random_ranged_attack`) ΓÇö `Source_Files/GameWorld/monsters.cpp`
- **Item/weapon behavior** (`swipe_nearby_items_fix`, `animate_items`, `m1_bce_pickup`) ΓÇö `Source_Files/GameWorld/items.cpp`, `weapons.cpp`
- **Platform dynamics** (`fix_sliding_on_platforms`, `m1_platform_flood`) ΓÇö `Source_Files/GameWorld/platforms.cpp`
- **Lua scripting** (`lua_increments_rng`, `lua_monster_killed_trigger_fix`) ΓÇö `Source_Files/Lua/lua_*.cpp`
- **Rendering** (`m1_landscape_effects`, `better_terminal_word_wrap`) ΓÇö `Source_Files/RenderMain/`, `RenderOther/`
- **Random number generation** (`lua_increments_rng`, `find_action_key_target_has_side_effects`) ΓÇö `Source_Files/GameWorld/world.cpp`

## Design Patterns & Rationale

**Version-Specific Configuration Registry:**
Rather than parameterizing every behavior with version checks (anti-pattern), FilmProfile.cpp centralizes all version-specific flags into named static structs. Each struct mirrors the previous version with only the differing flags changed, making evolutionary history explicit.

**Why this approach:**
- **Deterministic replay** ΓÇö Replays must execute identically across engine updates; querying a single global flag set is simpler than scattered version checks
- **Explicit version history** ΓÇö The struct sequence (alephone1_0 ΓåÆ ... ΓåÆ alephone1_11) documents which fixes were introduced when
- **Single point of switch** ΓÇö `load_film_profile()` is the only place version activation happens; no hidden state transitions elsewhere

**Tradeoffs:**
- **Global mutable state** ΓÇö Couples all consumers to this single global; modern engines prefer dependency injection or immutable config objects
- **Static initialization overhead** ΓÇö All 9 profiles are compiled in; unused profiles waste code space (negligible for 43 bools, but principle matters)
- **No runtime validation** ΓÇö A bogus `FilmProfileType` enum value produces undefined behavior (no default case, missing break statement)

## Data Flow Through This File

1. **Load phase** ΓåÆ Replay/film header is parsed (version extracted)
2. **Activation phase** ΓåÆ `load_film_profile(FilmProfileType)` is called with parsed version
3. **Switch dispatch** ΓåÆ A case statement selects the matching static profile and assigns it to global `film_profile`
4. **Query phase** (frame-by-frame) ΓåÆ GameWorld/Lua/Rendering subsystems read `film_profile.flag_name` at branching points
5. **Behavior divergence** ΓåÆ Conditional logic (e.g., `if (film_profile.ketchup_fix) { ... }`) implements version-specific paths

State is **write-once per replay** and **read-many per frame**, making this a classic ROM-like configuration pattern.

## Learning Notes

**What developers discover from this file:**
- How Marathon/Aleph One evolved across versions: Marathon 2 had minimal fixes; Infinity added weapon/AI tweaks; 1.0ΓÇô1.7 progressively patched physics, UI, and Lua edge cases
- The "fix" naming convention documents *what was broken*: `ketchup_fix`, `inexplicable_pin_change`, `m1_teleport_without_delay` are self-documenting bugfixes
- Backward compatibility is a **first-class design goal**, not an afterthought ΓÇö the replay system is core to the engine's identity
- Version profiles are frozen at release time and never modified (immutable once compiled), unlike live-config systems

**Idiomatic era quirks:**
- **Pre-modern C++ style** ΓÇö Static globals, no namespacing, manual version dispatch via switch (today you'd use a `std::map<FilmProfileType, FilmProfile>` or data-driven YAML)
- **No abstraction layer** ΓÇö Direct member access to `film_profile.flag_name` throughout the codebase (modern engines hide this behind a `VersionBehavior::query()` API)
- **Compile-time versioning** ΓÇö Versions are hardcoded; no runtime version registration (modern systems let plugins register versions dynamically)

## Potential Issues

1. **Missing `break` in ALEPH_ONE_1_7 case** (~line 362) ΓÇö Will cause fall-through to undefined behavior if that profile is selected (fall-through to `FILM_PROFILE_ALEPH_ONE_1_4` case or beyond, corrupting the profile state)
   
2. **No default case** ΓÇö If an invalid `FilmProfileType` is passed, `film_profile` is never updated, leaving stale state; should either assert or default to newest version

3. **No read barriers** ΓÇö Code throughout the engine reads `film_profile` without synchronization; if profiles are ever switched mid-frame (unlikely but architecturally possible), you could get torn reads or data races

4. **Silently ignore unknown versions** ΓÇö If a replay specifies a version not in this switch, the profile silently stays at its last value (should warn or error)
