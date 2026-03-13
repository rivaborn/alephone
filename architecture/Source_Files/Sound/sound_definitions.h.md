# Source_Files/Sound/sound_definitions.h

## File Purpose
Defines the data structures, constants, and static lookup tables that configure sound playback behavior in the Marathon/Aleph One game engine. It specifies how different sound categories (ambient, random, event-driven) should be attenuated by distance, obstruction, and pitch, and provides the binary format for loading sound resources from disk.

## Core Responsibilities
- Define sound behavior categories (quiet, normal, loud) and their depth-based attenuation curves
- Declare flags controlling sound restart, pitch, and obstruction behavior
- Define the binary sound file format (header and per-sound definition blocks)
- Initialize static lookup tables for ambient and random sound definitions with sound codes
- Specify sound play probabilities and pitch variation ranges per sound
- Define per-sound metadata: permutation counts, offsets, memory pointers, and playback history

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `sound_behavior` | enum | Categories of sound behavior (quiet, normal, loud) |
| `ambient_sound_definition` | struct | Minimal wrapper holding a single sound_index for ambient sounds |
| `random_sound_definition` | struct | Minimal wrapper holding a single sound_index for random event sounds |
| `sound_file_header` | struct | Binary file header for sound resource files; declares version, tag, source count, sound count |
| `sound_definition` | struct | Core per-sound record: code, behavior, flags, pitch range, permutations, file offsets, memory pointer |
| `depth_curve_definition` | struct | Distance-based volume curve: max/min volume and their associated distances |
| `sound_behavior_definition` | struct | Pair of depth curves (obstructed and unobstructed) for a sound behavior category |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sound_behavior_definitions` | `sound_behavior_definition[3]` | static | Lookup table for the 3 behavior categories (quiet, normal, loud); maps behavior to obstructed/unobstructed attenuation curves |
| `ambient_sound_definitions` | `ambient_sound_definition[28]` | static | Lookup table for 28 ambient sounds (water, wind, machinery, etc.); indexed by `_ambient_snd_*` enum |
| `random_sound_definitions` | `random_sound_definition[5]` | static | Lookup table for 5 random event sounds (drips, explosions, owl); indexed by `_random_snd_*` enum |

## Key Functions / Methods
None. This file is pure data definition and initialization. No functions or methods are defined here; all logic resides in SoundManager.

## Control Flow Notes
This file has no control flow. It is loaded at compile time and provides static configuration data. The sound manager consumes these definitions at initialization to set up audio playback behavior, distance attenuation, and probability thresholds. The commented-out `sound_definitions` array (lines ~240ΓÇô400) was superseded by dynamic loading from sound resource files, as indicated by the `sound_file_header` format and pointer fields in `sound_definition`.

## External Dependencies
- **`SoundManagerEnums.h`**: Provides `NUMBER_OF_AMBIENT_SOUND_DEFINITIONS`, `NUMBER_OF_RANDOM_SOUND_DEFINITIONS`, `NUMBER_OF_SOUND_DEFINITIONS`, `MAXIMUM_SOUND_VOLUME`, and sound code enums (`_snd_*`, `_ambient_snd_*`, `_random_snd_*`)
- **`world.h`**: Provides `WORLD_ONE` (world distance unit) for attenuation curve thresholds
- **Implicit**: `FOUR_CHARS_TO_INT()` macro (defined elsewhere, used in `SOUND_FILE_TAG`)
