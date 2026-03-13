# Source_Files/GameWorld/dynamic_limits.h

## File Purpose
Header file defining the interface for querying and configuring dynamic resource limits in the game engine. Supports limits on objects, NPCs, projectiles, effects, and collision structuresΓÇöall initialized from XML configuration and retrievable at runtime.

## Core Responsibilities
- Define enumeration of configurable entity limit types (objects, monsters, projectiles, effects, collision buffers, ephemera, garbage)
- Declare XML parser to load limit values from configuration
- Provide accessor function to retrieve current limit by type
- Support reset of limits to initial defaults

## Key Types / Data Structures
None (enum only; concrete data structures in implementation).

| Name | Kind | Purpose |
|------|------|---------|
| (enum, no name) | `enum` | Limit type constants: `_dynamic_limit_*` identifiers for each resource category |

## Global / File-Static State
None (state managed in implementation file).

## Key Functions / Methods

### parse_mml_dynamic_limits
- **Signature:** `void parse_mml_dynamic_limits(const InfoTree& root)`
- **Purpose:** Parse XML configuration tree and initialize dynamic limits from MML (Marathon Markup Language) data.
- **Inputs:** `root` ΓÇô XML/InfoTree object containing limit definitions
- **Outputs/Return:** None (void)
- **Side effects:** Populates internal limit storage; called during initialization
- **Calls:** Not visible in header
- **Notes:** Replaced resource-fork initialization (pre-May 2000); now XML-driven

### reset_mml_dynamic_limits
- **Signature:** `void reset_mml_dynamic_limits()`
- **Purpose:** Reset all dynamic limits to their default/initial values.
- **Inputs:** None
- **Outputs/Return:** None (void)
- **Side effects:** Clears custom limit overrides; restores defaults
- **Calls:** Not visible in header
- **Notes:** Useful for level/scenario resets or configuration reload

### get_dynamic_limit
- **Signature:** `uint16 get_dynamic_limit(int which)`
- **Purpose:** Query the current limit for a specific entity/resource type.
- **Inputs:** `which` ΓÇô limit type enum (e.g., `_dynamic_limit_objects`)
- **Outputs/Return:** `uint16` ΓÇô current limit value for that type
- **Side effects:** None
- **Calls:** Not visible in header
- **Notes:** Fast inline accessor for runtime constraint checks

## Control Flow Notes
Initialization phase: `parse_mml_dynamic_limits()` called once at engine startup to load XML config.  
Runtime: `get_dynamic_limit()` polled by world/entity managers to enforce spawn/active entity caps.  
Reset: `reset_mml_dynamic_limits()` called when loading new scenario or campaign level.

## External Dependencies
- `#include "cstypes.h"` ΓÇô standard integer types (`uint16`, `int`)
- Forward declaration: `class InfoTree` ΓÇô XML/config tree structure (defined elsewhere; likely in a config/parsing module)
