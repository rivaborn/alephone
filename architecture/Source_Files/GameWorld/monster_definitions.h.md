# Source_Files/GameWorld/monster_definitions.h

## File Purpose
Defines the data structures and enumerated constants that describe all monster types in the game engine. Contains class/faction relationships, behavioral flags, and a complete array of 40+ monster type definitions initialized with gameplay parameters (vitality, sounds, attacks, visual properties, etc.). The file serves as the authoritative data blueprint for monster instantiation and behavior.

## Core Responsibilities
- Define monster class enumerations and faction relationships (friends/enemies bitfields)
- Define monster behavioral flags (omniscience, invisibility, berserk, kamikaze, etc.)
- Define secondary constants: intelligence levels, door retry masks, and movement speeds
- Define `attack_definition` and `monster_definition` structures
- Provide complete initialization data for `original_monster_definitions` array (immutable master copy)
- Declare pack/unpack functions for serialization of monster definitions

## Key Types / Data Structures
| Name | Kind (struct/enum/class/typedef/interface/trait) | Purpose |
|------|------|---------|
| `attack_definition` | struct | Specifies a single attack (melee or ranged): projectile type, repetitions, aim error, range, keyframe trigger, and 3D firing offset (dx, dy, dz) |
| `monster_definition` | struct | Complete monster type profile: vitality/immunities/weaknesses, class/faction, sounds, shapes (stationary/moving/dying), visual range/arc, speed/gravity, two attack slots (melee & ranged), shrapnel behavior (~156 bytes) |

## Global / File-Static State
| Name | Type | Scope (global/static/singleton) | Purpose |
|------|------|-------|---------|
| `monster_definitions` | `monster_definition[NUMBER_OF_MONSTER_TYPES]` | global | Mutable array of current monster definitions; may be modified by MML/scripting at runtime |
| `original_monster_definitions` | `const monster_definition[NUMBER_OF_MONSTER_TYPES]` | global | Read-only master copy initialized with ~2900 lines of hardcoded monster data; serves as reference |

## Key Functions / Methods
None. This file contains only data structures, enumerations, and macro definitions. Runtime behavior is implemented in `monsters.c`.

**Trivial helpers (documented as macros):**
- `TYPE_IS_NEUTRAL()`, `TYPE_IS_ENEMY()`, `TYPE_IS_FRIEND()` ΓÇö bitfield class relationship checks
- Various flag test/set macros (e.g., `_monster_is_omniscient`, `_monster_flys`, `_monster_is_alien`) ΓÇö see enum in "enum /* flags */" section

## Control Flow Notes
Not applicable. This is a data definition file with no execution flow. Monster definitions are loaded at engine startup and referenced during:
1. **Monster instantiation** ΓÇö when a new monster is created in a level, its type index selects the corresponding `monster_definition`
2. **AI/behavior dispatch** ΓÇö during each frame, monster runtime code queries definition fields (speed, range, visual acuity, attack shapes, etc.)
3. **Serialization** ΓÇö when saving/loading game state, pack/unpack functions convert definitions to/from byte streams

## External Dependencies
- **Notable includes / imports:**
  - `effects.h` ΓÇö effect type constants (`_effect_fighter_blood_splash`, etc.) for impact effects
  - `items.h` ΓÇö item type constants for monster drops (`_i_magnum_magazine`, `_i_plasma_magazine`, etc.)
  - `map.h` ΓÇö type definitions: `world_distance`, `world_point3d`, `damage_definition`, shape descriptors
  - `monsters.h` ΓÇö external interface (not included but referenced in comments and prototypes)
  - `projectiles.h` ΓÇö projectile type constants used in `attack_definition` 
  - `SoundManagerEnums.h` ΓÇö sound ID constants (`_snd_fighter_activate`, `_snd_tick_chatter`, etc.)

- **Symbols defined elsewhere:**
  - `NUMBER_OF_MONSTER_TYPES` ΓÇö defined in `monsters.h`
  - `BUILD_COLLECTION()`, `FLAG()`, `UNONE`, `NONE` ΓÇö world/shape macros (likely from `map.h` or `world.h`)
  - World unit constants: `WORLD_ONE`, `WORLD_ONE_HALF`, `WORLD_ONE_FOURTH`, `WORLD_ONE_HALF`, `FIXED_ONE`, etc.
