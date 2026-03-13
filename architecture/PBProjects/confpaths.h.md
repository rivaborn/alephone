# PBProjects/confpaths.h

## File Purpose
Configuration header defining platform-specific package data directory paths. Currently disabled; intended to specify the resource file location for Aleph One (likely a game engine) on macOS.

## Core Responsibilities
- Define `PKGDATADIR` preprocessor constant for resource file paths (currently inactive)

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| PKGDATADIR | `#define` macro (commented) | File-static | macOS app bundle resource directory path |

## Key Functions / Methods
None.

## Control Flow Notes
Not inferable from this file. The single macro is commented out, so it has no active role in initialization or resource loading. If enabled, it would likely be included early in build configuration or resource initialization to point the engine toward game data files in the app bundle.

## External Dependencies
- None (preprocessor directive only; no includes or external symbols)

**Notes:** The commented state suggests this path definition was either superseded by runtime path discovery, moved to a different configuration file, or disabled during a refactor. The path `AlephOneSDL.app/Contents/Resources/DataFiles/` is characteristic of macOS app bundle layouts, indicating this was part of a cross-platform build system where macOS paths were explicitly configured.
