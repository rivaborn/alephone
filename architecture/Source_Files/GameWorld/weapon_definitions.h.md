# Source_Files/GameWorld/weapon_definitions.h

## File Purpose
Defines all weapon properties, configurations, and constants for the game engine. Provides data structures for weapon mechanics including triggers, animations, shell casings, and sound effects. Contains both the definition schema and hardcoded definitions for all player weapons.

## Core Responsibilities
- Define weapon classification system (melee, normal, dual-function, two-fisted, multipurpose)
- Enumerate weapon behavioral flags (automatic, overload, disappears after use, etc.)
- Define animation state indices for weapon in-hand rendering and shell casings
- Specify trigger mechanics (rounds per magazine, ammo type, firing rate, charging, recoil)
- Specify weapon visual & audio properties (firing light intensity, shape indices, sounds)
- Provide complete weapon definition array with per-weapon configuration for all game weapons
- Support serialization/deserialization of weapon data for save/load

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `shell_casing_definition` | struct | Physics parameters for ejected shell casings (position, velocity, acceleration) |
| `trigger_definition` | struct | Per-trigger weapon mechanics (magazine size, ammo type, fire rate, recoil, sounds, projectile type) |
| `weapon_definition` | struct | Complete weapon configuration (class, flags, visuals, animations, timing, both trigger definitions) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `shell_casing_definitions` | `shell_casing_definition[5]` | static | Physics data for 5 shell casing types (assault rifle, pistol center/left/right, SMG) |
| `weapon_ordering_array` | `int16[]` | static | Defines preferred weapon-selection order (fist, pistol, plasma, shotgun, assault rifle, SMG, flamethrower, launcher, alien, ball) |
| `weapon_definitions` | `weapon_definition[10]` | static | Runtime copy of weapon definitions (initialized from `original_weapon_definitions`) |
| `original_weapon_definitions` | `const weapon_definition[10]` | const | Hardcoded definitions for all 10 weapons (fist, magnum, plasma pistol, assault rifle, missile launcher, flamethrower, alien shotgun, shotgun, ball, SMG) |

## Key Functions / Methods

### unpack_weapon_definition
- Signature: `uint8 *unpack_weapon_definition(uint8 *Stream, weapon_definition *Objects, size_t Count)`
- Purpose: Deserialize weapon definitions from a binary stream (for loading saved games or map data)
- Inputs: byte stream pointer, weapon definition objects array, count of objects to unpack
- Outputs/Return: updated stream pointer (position after read)
- Side effects: modifies Objects array in-place
- Calls: (defined elsewhere; likely in weapons.c)
- Notes: Needed for "1-2-3 Converter" compatibility mentioned in comment

### pack_weapon_definition
- Signature: `uint8 *pack_weapon_definition(uint8 *Stream, weapon_definition *Objects, size_t Count)`
- Purpose: Serialize weapon definitions to a binary stream (for saving games or map data)
- Inputs: byte stream pointer, weapon definition objects array, count of objects to pack
- Outputs/Return: updated stream pointer (position after write)
- Side effects: writes to stream
- Calls: (defined elsewhere; likely in weapons.c)
- Notes: Counterpart to unpack function

## Control Flow Notes

This file is part of the **initialization phase**: weapon definitions are loaded once at startup and remain constant throughout gameplay. The `weapon_definitions` array is copied from `original_weapon_definitions` during weapon manager initialization. 

During **frame/update**: weapon mechanics reference these definitions (fire rate from `ticks_per_round`, ammo consumption from `rounds_per_magazine`, projectile type from `projectile_type`, sound playback from sound indices).

During **save/load**: serialization functions pack/unpack weapon state to disk, allowing game progress to persist.

## External Dependencies

**Notable includes:**
- `items.h` ΓÇö for item type constants (`_i_knife`, `_i_magnum_magazine`, etc.)
- `map.h` ΓÇö for `world_distance` type and `TICKS_PER_SECOND` constant
- `projectiles.h` ΓÇö for projectile type constants (`_projectile_pistol_bullet`, `_projectile_rocket`, etc.)
- `SoundManagerEnums.h` ΓÇö for sound constants (`_snd_magnum_firing`, `_snd_assault_rifle_shell_casings`, etc.)
- `weapons.h` ΓÇö for weapon enumeration constants and related structures

**External symbols used but not defined:**
- Type: `_fixed` (fixed-point arithmetic type, likely from map.h or global headers)
- Type: `world_distance` (distance measurement type)
- Constants: `FIXED_ONE`, `FIXED_ONE_HALF`, `FIXED_ONE/n` (fixed-point arithmetic macros)
- Constants: Sound indices (e.g., `_snd_magnum_firing`) and projectile types (e.g., `_projectile_pistol_bullet`)
- Constants: Item type indices (e.g., `_i_magnum`, `_i_assault_rifle_magazine`)
