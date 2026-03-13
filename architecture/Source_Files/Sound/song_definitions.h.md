# Source_Files/Sound/song_definitions.h

## File Purpose
Defines the data structures and constants for music/song management in the Aleph One game engine. Contains format definitions for song segments (introduction, chorus, trailer) and a static array of song instances for the game's soundtrack.

## Core Responsibilities
- Define `sound_snippet` struct for marking playback boundaries (start/end offsets)
- Define `song_definition` struct for complete song layout with multi-segment structure
- Declare song behavior flags (`_song_automatically_loops`)
- Provide `RANDOM_COUNT` macro for variable chorus repetitions
- Initialize a static array of song definitions for the engine

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `sound_snippet` | struct | Represents a contiguous audio segment with byte offsets (start/end) |
| `song_definition` | struct | Complete song structure: segments, playback flags, timing, and chorus repetition logic |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `songs[]` | `struct song_definition[]` | global | Static array of song definitions loaded at startup; indexed by song ID (inferred) |

## Key Functions / Methods
None. This is a data definition header.

## Control Flow Notes
Not applicableΓÇöthis file is declarative. The `songs` array is read by audio management code elsewhere in the engine. Song structure suggests a playback model: **introduction ΓåÆ repeated chorus segment(s) ΓåÆ trailer**, with optional auto-looping. The `restart_delay` (30 seconds in the example) likely controls pause duration between cycles.

## External Dependencies
- `csmisc.h` ΓÇö provides `MACHINE_TICKS_PER_SECOND` constant (1000)
- `cstypes.h` (indirect) ΓÇö for fixed-width integer types (`int16`, `int32`)
