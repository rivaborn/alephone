# Source_Files/Misc/achievements.h

## File Purpose
Header file defining the `Achievements` singleton class, which manages game achievements collection and integration with Lua scripting. Part of the game engine's reward/progression system.

## Core Responsibilities
- Provides singleton access to the achievements system via `instance()`
- Awards achievements by key identifier
- Exports achievements state to Lua for script integration
- Tracks and reports system disabled status with reasons

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| Achievements | class | Singleton manager for collecting and tracking game achievements |

## Global / File-Static State
None.

## Key Functions / Methods

### instance()
- Signature: `static Achievements* instance()`
- Purpose: Returns the singleton instance of the Achievements manager
- Inputs: None
- Outputs/Return: Pointer to Achievements singleton
- Side effects: None visible in header
- Calls: Not defined in this file
- Notes: Standard singleton pattern; implementation location unknown

### get_lua()
- Signature: `std::string get_lua()`
- Purpose: Exports current achievements state as Lua code/data representation
- Inputs: None
- Outputs/Return: String containing Lua-compatible representation
- Side effects: None visible
- Calls: Not defined in this file
- Notes: Used for scripting engine integration

### set()
- Signature: `void set(const std::string& key)`
- Purpose: Awards/unlocks an achievement by its identifier key
- Inputs: `key` ΓÇô achievement identifier
- Outputs/Return: void
- Side effects: Modifies internal achievement state
- Calls: Not defined in this file
- Notes: Implementation likely validates key existence and handles duplicates

### Accessors (trivial)
- `set_disabled_reason()` / `get_disabled_reason()` ΓÇô inline getters/setters for `disabled_reason` member

## Control Flow Notes
Not inferable from this file. This is an interface definition; frame/update patterns and achievement check logic reside in the implementation file.

## External Dependencies
- `<cstdint>` ΓÇô included but unused in this header
- `<string>` ΓÇô std::string for keys and Lua export data
- Implementation file (not shown) contains singleton instance and method definitions
