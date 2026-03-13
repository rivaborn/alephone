# Source_Files/Files/tags.h

## File Purpose
Defines tags and typecodes for the WAD file format system used in Aleph One (a Marathon game engine). Tags serve as 4-character identifiers for different data structures (maps, objects, physics models, save game data), while typecodes identify file types at the OS level (scenarios, savegames, physics files, etc.). The typecode system abstracts platform-specific file type codes.

## Core Responsibilities
- Define `Typecode` enum for abstracting game file types (scenario, savegame, physics, shapes, sounds, images, preferences, music, etc.)
- Provide tag constants for WAD map structures (points, lines, sides, polygons, objects, light sources, platforms, terminals, etc.)
- Define tags for save/load game data (player, monsters, effects, projectiles, weapons, platforms, terminals, Lua state, etc.)
- Define tags for physics models (both current and Marathon 1 legacy versions)
- Define tags for embedded resources (shape patches, sound patches, scripts)
- Define tags for preferences storage (graphics, sound, network, input, environment, etc.)
- Manage platform-specific typecode mappings via accessor functions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Typecode | enum | Abstract file type identifiers mapping to platform-specific OS typecodes |

## Global / File-Static State
None (typecode storage defined elsewhere).

## Key Functions / Methods

### initialize_typecodes
- Signature: `void initialize_typecodes();`
- Purpose: Initialize typecode mappings from resource fork
- Inputs: None
- Outputs/Return: None
- Side effects: Populates global typecode mapping
- Calls: Not inferable from this file
- Notes: Platform-dependent initialization; called during engine startup

### get_typecode
- Signature: `uint32 get_typecode(Typecode which);`
- Purpose: Retrieve OS-specific typecode for an abstract file type
- Inputs: `which` ΓÇô Typecode enum value
- Outputs/Return: uint32 OS typecode
- Side effects: None
- Calls: Not inferable from this file
- Notes: Defaults to `'????'` for unrecognized typecodes (as of Aug 28, 2000)

### set_typecode
- Signature: `void set_typecode(Typecode which, uint32 _type);`
- Purpose: Set OS-specific typecode for an abstract file type
- Inputs: `which` ΓÇô Typecode enum value; `_type` ΓÇô new typecode value
- Outputs/Return: None
- Side effects: Modifies global typecode mapping
- Calls: Not inferable from this file
- Notes: Added Jul 4, 2002; enables runtime typecode configuration

## Control Flow Notes
Purely definitional; no direct control flow. Included by I/O and file-loading modules. The typecode functions bridge between engine's abstract file type concepts (`Typecode` enum) and platform-specific OS type codes, enabling portable save/load and scenario/physics file handling.

## External Dependencies
- **cstypes.h** ΓÇô `uint32`, `FOUR_CHARS_TO_INT()` macro, `NONE` constant
- **filetypes_macintosh.c** ΓÇô Contains actual typecode value mappings (referenced in header comment, Feb 6, 2000)
- **\<vector\>** ΓÇô included but unused in this file
