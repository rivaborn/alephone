# Source_Files/CSeries/FilmProfile.h

## File Purpose

Defines configuration profiles that control engine behavior during film playback in Aleph One (Marathon engine recreation). Each profile contains boolean flags to enable/disable specific behavior quirks and bug fixes, allowing films recorded with different game versions to play back correctly.

## Core Responsibilities

- Define `FilmProfile` struct with ~45+ boolean flags for version-specific engine behaviors
- Enumerate supported film profile types (Marathon 2, Infinity, Aleph One 1.0ΓÇô1.7, etc.)
- Declare global `film_profile` instance to hold active profile
- Declare `load_film_profile()` function to switch between profiles

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| FilmProfile | struct | Container for boolean flags controlling engine behavior (physics, damage, spawning, UI, etc.) |
| FilmProfileType | enum | Identifies which film profile version to load |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| film_profile | FilmProfile | extern global | Active film profile instance loaded at runtime |

## Key Functions / Methods

### load_film_profile
- **Signature:** `void load_film_profile(FilmProfileType type)`
- **Purpose:** Load a film profile by type, configuring all engine behavior flags for the specified game version
- **Inputs:** `FilmProfileType type` ΓÇö enum value identifying which profile to activate
- **Outputs/Return:** None (sets global `film_profile`)
- **Side effects:** Modifies global `film_profile` state
- **Calls:** Not inferable from header
- **Notes:** Comment warns to update `interface.cpp` when adding new profile types

## Control Flow Notes

This file is used during film loading/initialization. When a film is about to play, `load_film_profile()` is called with the appropriate type to configure engine behaviors. The boolean flags in `film_profile` are then checked throughout the engine to conditionally apply version-specific behavior (physics calculations, damage models, spawning rules, etc.) so films play back identically to their original versions.

## External Dependencies

- None visible in this header (standard C struct/enum definitions only)
- Implementation of `load_film_profile()` likely defined in a `.c` file; callers use this header to access the global `film_profile` instance
