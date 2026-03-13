# Source_Files/Misc/achievements.cpp - Enhanced Analysis

## Architectural Role

The achievements system acts as a lightweight **validation and delegation layer** between the game engine and Steam, with security-first design preventing cheating in modded content. It sits in the Misc subsystem but deeply integrates with GameWorld (map state), Files (CRC validation), and Lua (trigger execution), forming a specialized pipeline: map validation ΓåÆ Lua code injection ΓåÆ achievement unlocking. Unlike most engine systems, it's **asymmetrically constrained** to single-player only, reflecting the asymmetry of fairness concerns between PvE and networked play.

## Key Cross-References

### Incoming (who depends on this file)
- **Lua scripting subsystem** ΓåÆ calls `Achievements::set()` from achievement trigger callbacks when gameplay conditions are met (likely hooked via Lua bindings in `lua/lua_map.cpp` or similar)
- **Game lifecycle / Interface** ΓåÆ calls `Achievements::instance()->get_lua()` during level load (probable call site: `marathon2.cpp`'s level transition code) to inject map-specific achievement Lua
- **Shell/Interface layer** ΓåÆ may display disabled achievement status in UI (via `set_disabled_reason()` side effects)

### Outgoing (what this file depends on)
- **Files subsystem** (`crc.h`) ΓåÆ `calculate_data_crc()`, `calculate_crc_for_file()` for physics/map integrity
- **GameWorld subsystem** (`map.h`) ΓåÆ `get_current_map_checksum()` for polygon/geometry integrity
- **GameWorld subsystem** (game mode check) ΓåÆ `get_game_controller() == _single_player` gate
- **Misc subsystem** (`Logging.h`) ΓåÆ `logNote()` for validation debugging
- **Steam integration layer** (`steamshim_child.h`) ΓåÆ `STEAMSHIM_setAchievement()`, `STEAMSHIM_storeStats()` (conditional on `HAVE_STEAM`)
- **Preferences subsystem** (`preferences.h`) ΓåÆ included but not directly used in this file (likely for future configuration)

## Design Patterns & Rationale

**Singleton without thread safety** (lines 13ΓÇô18) ΓÇö Classic pre-STL C++ pattern. Safe here because:
- Achievements are read-only after first call (deterministic per session)
- Game loop is single-threaded at policy level (no concurrent level loads)
- Allocation happens early in startup, never deallocated

**Compile-time Lua embedding via `#include`** (lines 45ΓÇô76) ΓÇö Unusual choice: embeds `.lua` as string literals rather than loading at runtime. Rationale:
- Reduces file I/O during level load (performance)
- Ensures achievements cannot be modified by modders (integrity)
- Couples achievement definitions to binary version (version-locked security)

**Checksum-based validation chain** (lines 33ΓÇô41, 52ΓÇô60, 73) ΓÇö Two-layer integrity check:
1. **Map checksum** discriminates between Marathon 1, M2, M2 Win95, Infinity (hardcoded constants)
2. **Physics checksum** ensures no third-party physics mods (prevents custom behavior farms)
- Empty Lua return on mismatch is silent disable (no exception), allowing graceful degradation

**Single-player gating** (line 25) ΓÇö Achievements disabled entirely in multiplayer. Reflects trust boundary: single-player state is local/trustworthy; networked play requires server-side validation (out of scope here).

## Data Flow Through This File

```
[Level Load] 
  Γåô calls Achievements::get_lua()
  Γö£ΓöÇ checks game_controller == single-player? (gate: if false, return "")
  Γö£ΓöÇ fetches map_checksum, physics_checksum from GameWorld
  Γö£ΓöÇ switches on map_checksum (M1/M2/Inf)
  Γöé  Γö£ΓöÇ validates physics_checksum matches expected
  Γöé  ΓööΓöÇ includes map-specific .lua file (m1_achievements.lua, etc.) as string
  Γö£ΓöÇ on checksum mismatch: calls set_disabled_reason() (state mutation)
  Γö£ΓöÇ on success: returns Lua code
  ΓööΓöÇ logs diagnostics (logNote)

[Lua Engine Execution]
  Γåô Lua receives code, registers achievement triggers
  Γåô Game loop continues, Lua callbacks fire on game events

[Achievement Unlock]
  Γåô Lua callback calls Achievements::set(key)
  Γö£ΓöÇ logs the unlock attempt (logNote)
  Γö£ΓöÇ if HAVE_STEAM: calls STEAMSHIM_setAchievement(key)
  Γö£ΓöÇ if HAVE_STEAM: calls STEAMSHIM_storeStats()
  ΓööΓöÇ returns (void, fire-and-forget)
```

**Key state**: Only `instance_` is file-static; validation results stored externally via `set_disabled_reason()` side effect.

## Learning Notes

- **Checksum-based integrity** is a pre-DRM, pre-online-validation pattern common in early 2000s offline games. Modern engines delegate this to server-side achievement servers.
- **Preprocessor embedding** of Lua is unusual; most modern engines use string resources or embedded filesystems (e.g., PHYSFS). This reflects Aleph One's retro architecture.
- **Lazy singleton** shows no dependency injection; would be challenging to unit-test or mock in a modern codebase.
- **Single-player-only achievements** reflects the era's assumption that single-player cheating is acceptable (cosmetic consequence) vs. multiplayer unfairness (competitive harm).

## Potential Issues

1. **Thread-safety**: Singleton without mutex. If startup/level-load races with Lua callbacks, `instance_` allocation or `get_lua()` result could be corrupted. Low risk in practice (game loop is sequential), but a `volatile` or `std::once_flag` would be safer.

2. **Silent failure**: Checksum mismatch returns empty Lua (achievement disabled) with only logging. If log doesn't appear, user has no feedback. Could add `alert_user()` or UI indicator.

3. **Hardcoded checksums**: Tightly couples achievements to exact shipped assets. Any future map repack/rebuild breaks achievements. No versioning mechanism.

4. **Lua embedding ordering**: `#include` of `.lua` files must occur at compile time; if `.lua` files are malformed or missing at build, linker errors occur. No runtime fallback.
