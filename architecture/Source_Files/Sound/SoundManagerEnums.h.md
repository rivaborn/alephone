# Source_Files/Sound/SoundManagerEnums.h

## File Purpose
Centralized enumeration and constant definitions for the sound manager system. Extracted from the main SoundManager header to reduce header bloat and organize sound-related type definitions and identifiers used throughout the engine.

## Core Responsibilities
- Define unique identifiers for ~200+ sound effects (weapons, ambient, entity, interactive)
- Define ambient and random sound type enumerations
- Provide audio format and channel configuration constants
- Define initialization flags for sound manager configuration (bitmask-based)
- Define sound obstruction and acoustic property flags
- Define frequency adjustment constants for Doppler and sound modulation effects

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| (unnamed) | enum | Ambient sound codes (_ambient_snd_water, _ambient_snd_wind, etc.) ΓÇö 29 distinct types |
| (unnamed) | enum | Random ambient sound codes (_random_snd_water_drip, _random_snd_owl, etc.) ΓÇö 5 distinct types |
| (unnamed) | enum | Primary sound definitions (_snd_startup, _snd_teleport_in, etc.) ΓÇö 200+ sound IDs organized by category |
| (unnamed) | enum | Sound volume configuration (NUMBER_OF_SOUND_VOLUME_LEVELS = 8, MAXIMUM_SOUND_VOLUME = 256) |
| (unnamed) | enum | Sound source types (_8bit_22k_source, _16bit_22k_source) |
| AudioFormat | enum class | Audio bit depth (_8_bit, _16_bit, _32_float) |
| ChannelType | enum class | Speaker configuration (_mono, _stereo, _quad, _5_1, _6_1, _7_1) |
| (unnamed) | enum | Initialization flags (bitmask: _dynamic_tracking_flag, _doppler_shift_flag, _16bit_sound_flag, etc.) ΓÇö 11 flags |
| (unnamed) | enum | Sound obstruction flags (bitmask: _sound_was_obstructed, _sound_was_media_obstructed, _sound_was_media_muffled) |
| (unnamed) | enum | Frequency multipliers (_lower_frequency, _normal_frequency, _higher_frequency, _m1_high_frequency) |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a definitions-only header.

## Control Flow Notes
Not applicable. This header provides compile-time constants and type definitions used by sound manager initialization and runtime sound playback. The enums serve as configuration and indexing identifiers rather than control flow points.

## External Dependencies
- **cstypes.h**: Provides fixed-point type (`_fixed`), fixed-point constants (`FIXED_ONE`, `FIXED_ONE_HALF`), and integer type definitions (uint8, int32, etc.)

## Notes
- Sound definitions are organized semantically (weapons, entities, interactions, ambient, media effects)
- Bitmask enums (initialization flags, obstruction flags) suggest bit-level feature control
- Frequency values computed relative to `FIXED_ONE` (16.16 fixed-point)
- Multiple enum naming conventions: legacy C-style (unnamed enums with _prefix members) and modern C++ (enum class AudioFormat, ChannelType)
