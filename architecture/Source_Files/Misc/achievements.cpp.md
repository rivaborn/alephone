# Source_Files/Misc/achievements.cpp

## File Purpose
Implements the achievements system for Aleph One using a singleton pattern. Manages achievement validation, Lua-based achievement trigger definitions, and Steam integration for unlocking achievements during gameplay.

## Core Responsibilities
- Provides singleton accessor for global achievement management
- Validates map and physics file checksums to prevent achievements in modified/custom content
- Generates and returns Lua code defining achievement triggers and conditions per map
- Posts achievement unlocks to Steam (when `HAVE_STEAM` is defined)
- Records disabled achievements with explanatory reasons (e.g., third-party physics detected)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Achievements` | class | Singleton managing achievement state and Steam integration |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `instance_` | `Achievements*` | static (function-scoped) | Singleton instance holder in `instance()` |

## Key Functions / Methods

### instance()
- **Signature:** `static Achievements* instance()`
- **Purpose:** Provides lazy-initialized singleton accessor for the achievements system
- **Inputs:** None
- **Outputs/Return:** Pointer to the single `Achievements` instance
- **Side effects (global state, I/O, alloc):** Allocates singleton on first call
- **Calls (direct calls visible in this file):** None visible
- **Notes:** Classic C++ singleton pattern; not thread-safe (no mutex)

### get_lua()
- **Signature:** `std::string get_lua()`
- **Purpose:** Returns achievement trigger definitions as Lua code, conditional on current map validity
- **Inputs:** None (reads global game state via `get_game_controller()`, `get_current_map_checksum()`, `get_physics_file_checksum()`)
- **Outputs/Return:** Lua code string (empty if validation fails or not single-player)
- **Side effects (global state, I/O, alloc):** Calls `set_disabled_reason()` on validation failure; logs to engine logging system
- **Calls (direct calls visible in this file):**
  - `get_game_controller()` ΓÇô checks if single-player mode
  - `get_current_map_checksum()`, `get_physics_file_checksum()` ΓÇô fetch file checksums
  - `set_disabled_reason()` ΓÇô records why achievements are disabled
  - `logNote()` ΓÇô logs checksum mismatches
  - `#include` directives for `m1_achievements.lua`, `m2_achievements.lua`, `inf_achievements.lua`
- **Notes:** 
  - Embeds Lua files via preprocessor `#include` (unusual but allows string inclusion)
  - Only enables achievements for **exact** map/physics combinations (Marathon 1, Marathon 2 / Win95, Infinity)
  - Disables achievements if physics file doesn't match (prevents cheating via mods)
  - Only active when `HAVE_STEAM` is defined and game is single-player

### set()
- **Signature:** `void set(const std::string& key)`
- **Purpose:** Posts a named achievement unlock to Steam and persists stats
- **Inputs:** `key` ΓÇô achievement identifier (e.g., `"ACH_VICTORY"`)
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Calls Steam API to unlock achievement and store stats
- **Calls (direct calls visible in this file):**
  - `logNote()` ΓÇô logs the achievement post
  - `STEAMSHIM_setAchievement()` ΓÇô unlocks in Steam (if `HAVE_STEAM` defined)
  - `STEAMSHIM_storeStats()` ΓÇô persists achievement data
- **Notes:** Logs every achievement attempt for debugging; no-op if Steam not available

## Control Flow Notes
- **Initialization:** Singleton created on first call to `instance()`, likely early in game startup
- **Map Loading:** `get_lua()` called when entering a level; returns empty string if map/physics invalid ΓåÆ achievements disabled for that level
- **Gameplay:** Lua engine (defined in embedded `.lua` files) calls `set()` when achievement conditions met
- **Frame Loop:** Not directly invoked; Lua-driven via callbacks from game state changes (damage, monster kills, etc.)

## External Dependencies
- **Optional:** `steamshim_child.h` ΓÇô Steam shim API (only if `HAVE_STEAM` defined)
- **Required:**
  - `crc.h` ΓÇô `get_physics_file_checksum()` 
  - `extensions.h` ΓÇô physics file handling
  - `Logging.h` ΓÇô `logNote()` macro
  - `map.h` ΓÇô `get_current_map_checksum()`, game mode checks
  - `preferences.h` ΓÇô likely for player/environment prefs (not used in this file directly)
- **Defined elsewhere:** `get_game_controller()`, `get_current_map_checksum()`, `get_physics_file_checksum()`
