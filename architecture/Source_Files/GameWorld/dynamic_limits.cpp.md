# Source_Files/GameWorld/dynamic_limits.cpp

## File Purpose
Manages runtime-configurable limits for game world entity pools (objects, monsters, effects, projectiles, etc.). Provides three preset configurations (Marathon 2 original, Aleph One 1.0 expanded, and 1.1 compatible) and allows override via MML XML markup. Coordinates reallocation of backing arrays in dependent subsystems when limits change.

## Core Responsibilities
- Maintain three static limit configurations for different game versions/compatibility modes
- Parse MML XML configuration to override individual entity limits
- Select and initialize appropriate limit preset based on film profile flags
- Reallocate entity storage arrays across subsystems when limits change
- Provide accessor function for runtime limit queries
- Ensure consistency between limit values and allocated array sizes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `m2_dynamic_limits` | uint16 array (static) | Original Marathon 2 defaults |
| `a1_1_0_dynamic_limits` | uint16 array (static) | Aleph One 1.0 expanded defaults |
| `a1_1_1_dynamic_limits` | uint16 array (static) | Aleph One 1.1 compatibility defaults |
| `dynamic_limits` | std::vector\<uint16\> | Active runtime limits (mutable) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `dynamic_limits_loaded` | bool | file-static | Tracks whether limits have been initialized; defaults to false |
| `dynamic_limits` | std::vector\<uint16\> | file-static | Holds current active limits; size = NUMBER_OF_DYNAMIC_LIMITS |

## Key Functions / Methods

### reset_dynamic_limits
- **Signature:** `static void reset_dynamic_limits()`
- **Purpose:** Initialize `dynamic_limits` from one of three preset arrays based on `film_profile` flags.
- **Inputs:** None (reads global `film_profile` object).
- **Outputs/Return:** None (modifies `dynamic_limits` and sets `dynamic_limits_loaded = true`).
- **Side effects:** Overwrites current `dynamic_limits` vector content; sets flag indicating limits are loaded.
- **Calls:** `dynamic_limits.assign()` (std::vector method).
- **Notes:** Respects compatibility hierarchy: checks `increased_dynamic_limits_1_1` first, then `increased_dynamic_limits_1_0`, otherwise uses Marathon 2 defaults. No reallocation happens here.

### reallocate_dynamic_limits
- **Signature:** `static void reallocate_dynamic_limits()`
- **Purpose:** Resize global entity storage arrays (effects, objects, monsters, projectiles, ephemera) based on current `dynamic_limits` values.
- **Inputs:** None (reads current `dynamic_limits` array).
- **Outputs/Return:** None.
- **Side effects:** Resizes `EffectList`, `ObjectList`, `MonsterList`, `ProjectileList` via `.resize()`; calls `allocate_pathfinding_memory()` and `allocate_ephemera_storage()` to allocate supporting memory.
- **Calls:** Direct calls to `EffectList.resize()`, `ObjectList.resize()`, `MonsterList.resize()`, `ProjectileList.resize()`, `allocate_pathfinding_memory()`, `allocate_ephemera_storage()`.
- **Notes:** Called after both reset and MML parse to sync allocated memory to configured limits.

### parse_limit_value
- **Signature:** `static void parse_limit_value(const InfoTree& root, std::string child, int type)`
- **Purpose:** Helper to parse a single MML XML element and update corresponding limit value.
- **Inputs:** 
  - `root`: InfoTree root node to search within.
  - `child`: XML child element name to match (e.g., "objects", "monsters").
  - `type`: Index into `dynamic_limits` array (e.g., `_dynamic_limit_objects`).
- **Outputs/Return:** None (modifies `dynamic_limits[type]`).
- **Side effects:** Updates `dynamic_limits[type]` if matching child element found.
- **Calls:** `root.children_named(child)` (InfoTree iteration), `limit.read_attr_bounded<uint16>("value", ...)` (InfoTree attribute parsing).
- **Notes:** Silently does nothing if no matching child element; bounds limit to [0, 32767].

### reset_mml_dynamic_limits
- **Signature:** `void reset_mml_dynamic_limits()`
- **Purpose:** Public entry point to reset limits to defaults and reallocate arrays.
- **Inputs:** None (reads `film_profile`).
- **Outputs/Return:** None.
- **Side effects:** Calls `reset_dynamic_limits()` and `reallocate_dynamic_limits()`.
- **Calls:** `reset_dynamic_limits()`, `reallocate_dynamic_limits()`.
- **Notes:** Used when reloading a map or initializing the game world.

### parse_mml_dynamic_limits
- **Signature:** `void parse_mml_dynamic_limits(const InfoTree& root)`
- **Purpose:** Public entry point to parse all dynamic limit overrides from MML configuration.
- **Inputs:** `root`: InfoTree root node containing MML limit definitions.
- **Outputs/Return:** None.
- **Side effects:** Updates all 11 entries in `dynamic_limits` based on XML; calls `reallocate_dynamic_limits()` to sync arrays.
- **Calls:** `parse_limit_value()` (11 times, once per limit type), `reallocate_dynamic_limits()`.
- **Notes:** Called during MML parsing phase; must be called after `reset_mml_dynamic_limits()` for proper initialization order.

### get_dynamic_limit
- **Signature:** `uint16 get_dynamic_limit(int which)`
- **Purpose:** Accessor to retrieve the current limit value for a given entity type.
- **Inputs:** `which`: Index (e.g., `_dynamic_limit_objects`, `_dynamic_limit_monsters`).
- **Outputs/Return:** uint16 limit value.
- **Side effects:** None.
- **Calls:** None (direct array access).
- **Notes:** Fallback behavior: if `dynamic_limits_loaded` is false, returns from `a1_1_1_dynamic_limits` as a safe default. This guards against queries before initialization.

## Control Flow Notes
**Initialization order:** Game startup calls `reset_mml_dynamic_limits()` to establish baseline limits from `film_profile`, then MML parsing calls `parse_mml_dynamic_limits()` to override. Both trigger `reallocate_dynamic_limits()` to ensure arrays match active limits. Subsequent `get_dynamic_limit()` queries during frame updates retrieve the configured values used by entity creation routines (monsters, objects, effects, projectiles managers).

## External Dependencies
- **Includes (defined elsewhere):**
  - `cseries.h` ΓÇö core series utilities and types
  - `map.h` ΓÇö map structures; defines `MAXIMUM_*_PER_MAP` macros that call `get_dynamic_limit()`
  - `effects.h`, `monsters.h`, `projectiles.h` ΓÇö entity subsystem headers
  - `ephemera.h` ΓÇö render effect particles
  - `flood_map.h` ΓÇö pathfinding support
  - `InfoTree.h` ΓÇö XML tree parser for MML
- **Global symbols used (defined elsewhere):**
  - `EffectList`, `ObjectList`, `MonsterList`, `ProjectileList` ΓÇö std::vector entity pools (from effects.cpp, map.cpp, monsters.cpp, projectiles.cpp)
  - `film_profile` ΓÇö global object tracking Marathon version compatibility flags
  - `allocate_pathfinding_memory()`, `allocate_ephemera_storage()` ΓÇö subsystem allocation routines
