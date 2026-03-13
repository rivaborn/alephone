# Source_Files/GameWorld/platform_definitions.h

## File Purpose
Defines default sound and behavioral configurations for all platform types in the game world. Provides a lookup table that maps each platform type (doors, platforms, etc.) to its associated sound effects, damage properties, and initial behavior flags. This file is part of the Aleph One engine (Marathon mod) and serves as initialization data for platform behavior.

## Core Responsibilities
- Enumerate platform sound codes (starting, stopping, obstructed, uncontrollable)
- Define the `platform_definition` struct to hold per-platform-type configuration
- Populate the `platform_definitions` array with concrete data for all 9 platform types
- Associate each platform type with its audio assets (opening, closing, ambient, obstruction sounds)
- Set default behavior flags for each platform type (speed, delay, control modes, damage)
- Provide damage characteristics (damage type and thresholds) for each platform type

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `platform_definition` | struct | Encapsulates all configurable properties for a single platform type, including sounds, key item dependencies, default static/dynamic data, and damage behavior |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `platform_definitions` | `platform_definition[NUMBER_OF_PLATFORM_TYPES]` | global | Lookup table (9 entries) that provides default configuration for each platform type: doors (Spht, Pfhor, heavy), platforms (standard and noisy variants) |

## Key Functions / Methods
None. This file contains only data definitions and constant declarations.

## Control Flow Notes
This is purely **data initialization**. The `platform_definitions` array is statically initialized at compile time and serves as a reference table during engine startup and runtime. The engine likely calls `get_defaults_for_platform_type()` (declared in `platforms.h`) to retrieve configuration from this table when instantiating new platforms in the game world.

## External Dependencies

### Includes
- `platforms.h` ΓÇö Provides enums for platform types (`_platform_is_spht_door`, `_platform_is_pfhor_door`, etc.), speed/delay constants, flag macros, and struct definitions (`static_platform_data`, `damage_definition`)
- `SoundManagerEnums.h` ΓÇö Provides sound ID enumerations (e.g., `_snd_spht_door_opening`, `_ambient_snd_spht_door`)

### Defined Elsewhere
- `static_platform_data` ΓÇö struct for platform defaults (speed, delay, flags, polygon index)
- `damage_definition` ΓÇö struct for damage properties (type, minimum damage, maximum damage, threshold)
- Sound enums (`_snd_*`, `_ambient_snd_*`) ΓÇö from SoundManagerEnums.h
- Platform type enums (`_platform_is_*`) and flag macros (`FLAG()`) ΓÇö from platforms.h
