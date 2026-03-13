# Source_Files/GameWorld/editor.h

## File Purpose
Header file defining version constants for the Aleph One game engine's map editor and data format compatibility. Tracks Marathon engine versions (One, Two, Infinity) to manage file format evolution.

## Core Responsibilities
- Define editor map data format version constants
- Provide version identifiers for Marathon engine iterations (1.0, 2.0, Infinity)
- Establish current editor map version by aliasing to latest supported format

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MARATHON_ONE_DATA_VERSION` | `#define` macro (0) | Global | Version identifier for Marathon 1 data format |
| `MARATHON_TWO_DATA_VERSION` | `#define` macro (1) | Global | Version identifier for Marathon 2 data format |
| `MARATHON_INFINITY_DATA_VERSION` | `#define` macro (2) | Global | Version identifier for Marathon Infinity data format |
| `EDITOR_MAP_VERSION` | `#define` macro | Global | Current editor map version; aliases to `MARATHON_INFINITY_DATA_VERSION` (2) |

## Key Functions / Methods
None.

## Control Flow Notes
Not applicable. This is a constant-definition header with no executable logic.

## External Dependencies
- Standard C header guard pattern (`__EDITOR_H_`)
- No external includes or dependencies
- Designed to be included by other game world editor modules that need version information

**Notes:**
- Version constant scheme (0, 1, 2) allows serialization and compatibility checks when loading/saving map files across different Marathon engine versions.
- Comment indicates `EDITOR_MAP_VERSION` was updated in Feb 2000 to support Infinity format.
