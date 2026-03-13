# Source_Files/CSeries/FilmProfile.cpp

## File Purpose
Defines version-specific compatibility and bug-fix configurations for different Marathon and Aleph One engine versions. Provides a mechanism to load the correct behavioral profile based on film/replay version.

## Core Responsibilities
- Declare static FilmProfile instances for eight engine versions (Marathon 2, Marathon Infinity, Aleph One 1.0ΓÇô1.7)
- Initialize a global mutable `film_profile` instance with a default value
- Provide a switch-based loader to activate the appropriate profile configuration

## Key Types / Data Structures
None defined here (FilmProfile struct defined in header).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `alephone1_11` | FilmProfile | static | Newest version profile with all recent fixes enabled |
| `alephone1_7` | FilmProfile | static | Aleph One 1.7 profile (4 flags disabled relative to 1.11) |
| `alephone1_4` | FilmProfile | static | Aleph One 1.4 profile (fewer fixes than 1.7) |
| `alephone1_3` | FilmProfile | static | Aleph One 1.3 profile |
| `alephone1_2` | FilmProfile | static | Aleph One 1.2 profile (fewer fixes) |
| `alephone1_1` | FilmProfile | static | Aleph One 1.1 profile (differs from 1.2 in ~3 flags) |
| `alephone1_0` | FilmProfile | static | Aleph One 1.0 baseline (most flags disabled/inverted) |
| `marathon2` | FilmProfile | static | Marathon 2 original profile (minimal fixes) |
| `marathon_infinity` | FilmProfile | static | Marathon Infinity profile (mid-way fixes) |
| `film_profile` | FilmProfile | global | Active profile used by engine; mutable |

## Key Functions / Methods

### load_film_profile
- Signature: `void load_film_profile(FilmProfileType type)`
- Purpose: Switch the active global `film_profile` to a version-specific configuration
- Inputs: FilmProfileType enum value (9 cases covering Aleph One 1.0ΓÇô1.7, Marathon 2, Marathon Infinity, and DEFAULT)
- Outputs/Return: None
- Side effects: Mutates global `film_profile` state
- Calls: None (direct assignment only)
- Notes: Switch covers 8 of 9 FilmProfileType enum values; `FILM_PROFILE_ALEPH_ONE_1_7` case lacks `break`, causing fall-through (likely unintended bug). Header comment warns maintainers to update interface.cpp when adding profiles.

## Control Flow Notes
This is a configuration module loaded at engine initialization or during film playback setup. No rendering, physics, or frame-by-frame logic hereΓÇöpurely a registry of compatibility flags that other subsystems query via the global `film_profile`.

## External Dependencies
- `#include "FilmProfile.h"` ΓÇö defines FilmProfile struct (43 boolean fields) and FilmProfileType enum
- No other includes or external symbols

---

**Note:** Missing `break` statement in the `FILM_PROFILE_ALEPH_ONE_1_7` case (line ~362) will cause unintended behavior if that profile is selected, falling through to undefined state. Recommend adding the missing `break`.
