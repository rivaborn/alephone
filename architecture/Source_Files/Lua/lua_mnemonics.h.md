# Source_Files/Lua/lua_mnemonics.h

## File Purpose
Defines string-to-integer mnemonic lookup tables for Lua scripting integration. Maps human-readable identifiers (e.g., "water", "missile") to numeric constant values across 42 game engine categories, enabling Lua scripts to reference game engine concepts using symbolic names.

## Core Responsibilities
- Define `lang_def` struct as a key-value pair container (string name ΓåÆ int32 value)
- Provide 42 global mnemonic arrays covering:
  - Audio (ambient sounds, sound effects)
  - Game mechanics (damage types, difficulty, game modes, scoring modes)
  - Game objects (items, weapons, monsters, projectiles, scenery, platforms)
  - Rendering (fade types, light states, transfer modes, textures, interface colors/fonts/rects)
  - Physics (media types, polygon types, control panel classes)
  - Player/monster state (colors, actions, modes, sensor blip types)
- Enable Lua scripts to reference engine enumerations by string names instead of raw numeric values

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `lang_def` | struct | Two-member struct: `const char *name` (mnemonic string) and `int32 value` (numeric constant); all arrays are null-terminated (final entry is `{0, 0}`) |

## Global / File-Static State
| Name (sample) | Type | Scope | Purpose |
|---|---|---|---|
| `Lua_AmbientSound_Mnemonics[]` | const `lang_def[]` | global | 29 ambient sound identifiers (water, lava, wind, doors, etc.) |
| `Lua_MonsterType_Mnemonics[]` | const `lang_def[]` | global | 47 monster/creature type identifiers (fighters, drones, cyborgs, etc.) |
| `Lua_Sound_Mnemonics[]` | const `lang_def[]` | global | 215 sound effect identifiers (weapons, footsteps, creature sounds, etc.) |
| `Lua_EffectType_Mnemonics[]` | const `lang_def[]` | global | 73 visual effect identifiers (explosions, splashes, teleports, etc.) |
| *40 other arrays* | const `lang_def[]` | global | ItemType, ProjectileType, SceneryType, InterfaceRect, InterfaceColor, etc. |

## Key Functions / Methods
None. This file contains only static data definitions.

## Control Flow Notes
This file serves an **initialization/configuration** role in the Lua integration layer. Its mnemonic arrays are likely loaded by the Lua script engine (invoked from `lua_script.h`) at startup to populate reverse-lookup tables or Lua tables, allowing scripts to use readable names like `"water"` instead of hardcoded numeric values like `0`. Arrays are terminated with null entries, supporting iteration-based initialization patterns.

## External Dependencies
- **`#include "lua_script.h"`** ΓÇô Declares Lua integration functions and enumerations (e.g., `_game_of_most_points`). The bundled header shows this module controls script loading, cleanup, and engine callbacks.
- **Copyright/License** ΓÇô Dual GPL v3 and author attribution (Gregory Smith, 2008).

## Notes
- **Design pattern**: All mnemonic arrays follow the same schema (nameΓÇôvalue pairs), making bulk-loading feasible.
- **Enum-like semantics**: Values are typically sequential (0, 1, 2, ΓÇª) or use bit flags (e.g., `MonsterClass` uses `0x0001`, `0x0002`, `0x0004`).
- **Minor typo**: `Lua_LightFunction_Mnenonics` should be `Mnemonic**s**` (missing 's').
- **Duplicate entry**: `Lua_Sound_Mnemonics` contains both `{"heavy shpt door", 144}` and `{"heavy spht door", 144}` at indices 143ΓÇô144 (likely a copy-paste error).
