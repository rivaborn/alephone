# Source_Files/GameWorld/effect_definitions.h

## File Purpose
Defines static configuration data for visual and audio effects that occur in the game world (explosions, blood splashes, teleportation, water/lava/sewage splashes, etc.). Maps each effect type to sprite animation properties and sound parameters.

## Core Responsibilities
- Define effect property struct and behavior flags
- Declare static effect definition database
- Store animation collection/shape mappings for all ~67 effect types
- Provide serialization functions for save/load operations
- Support effect variants for different media types (water, lava, sewage, goo, Jjaro)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `effect_definition` | struct | Encodes animation collection, shape index, sound pitch, flags, and timing delays for a single effect type |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `effect_definitions` | `effect_definition[]` | static | Mutable copy of effect definitions; populated from `original_effect_definitions` at init |
| `original_effect_definitions` | `const effect_definition[]` | static | Hardcoded effect data for all `NUMBER_OF_EFFECT_TYPES` effects (rockets, grenades, weapons, splashes, teleports, etc.) |

## Key Functions / Methods

### unpack_effect_definition
- Signature: `uint8 *unpack_effect_definition(uint8 *Stream, effect_definition *Objects, size_t Count)`
- Purpose: Deserialize effect definitions from a byte stream
- Inputs: Stream pointer, effect definition array, count
- Outputs/Return: Pointer to next byte in stream after unpacking
- Side effects: Populates `Objects` array from stream data
- Calls: Not visible in this file
- Notes: Part of save/load pipeline for game state

### pack_effect_definition
- Signature: `uint8 *pack_effect_definition(uint8 *Stream, effect_definition *Objects, size_t Count)`
- Purpose: Serialize effect definitions to a byte stream
- Inputs: Stream pointer, effect definition array, count
- Outputs/Return: Pointer to next byte in stream after packing
- Side effects: Writes to stream; does not modify `Objects`
- Calls: Not visible in this file
- Notes: Part of save/load pipeline for game state

## Control Flow Notes
This file contains only data declarations and function prototypes; no initialization logic resides here. The `original_effect_definitions` array is likely copied into `effect_definitions` at engine startup. Effects are instantiated elsewhere (via `new_effect()` in effects.c) using indices into these tables during gameplay events (weapon impacts, object splashes, teleportation).

## External Dependencies
- **effects.h**: Defines effect type enum (e.g., `_effect_rocket_explosion`, `_effect_teleport_object_in`), `NUMBER_OF_EFFECT_TYPES` constant, and prototypes for `new_effect()`, `update_effects()`, `remove_effect()`, serialization functions
- **map.h**: Provides constants like `TICKS_PER_SECOND`, `NONE` macro, and serialization infrastructure
- **SoundManagerEnums.h**: Provides sound index constants (e.g., `_snd_teleport_in`) and frequency constants (`_normal_frequency`, `_higher_frequency`, `_lower_frequency`)
