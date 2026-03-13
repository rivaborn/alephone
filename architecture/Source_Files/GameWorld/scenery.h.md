# Source_Files/GameWorld/scenery.h

## File Purpose
Header declaring functions for managing scenery (static/decorative objects) in the game world. Supports scenery creation, animation, damage, shape randomization, and data-driven configuration via MML. Part of the Aleph One/Marathon engine.

## Core Responsibilities
- Initialize and manage scenery object lifecycle
- Animate scenery and randomize visual variants
- Track and apply damage to scenery objects
- Retrieve scenery collection data (sprite/shape resources)
- Parse and reset MML-based scenery configuration
- Support Lua scripting for runtime scenery manipulation

## Key Types / Data Structures
None defined in this file.

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_scenery
- Signature: `void initialize_scenery(void);`
- Purpose: Initialize the scenery system (likely zeroing or preparing data structures)
- Inputs: None
- Outputs/Return: None
- Side effects: Sets up global scenery state
- Calls: Not inferable from this file
- Notes: Called during engine initialization

### new_scenery
- Signature: `short new_scenery(struct object_location *location, short scenery_type);`
- Purpose: Create and place a new scenery object in the world
- Inputs: Location (3D position + polygon), scenery type ID
- Outputs/Return: Object index (short)
- Side effects: Allocates scenery object, modifies world state
- Calls: Not inferable from this file
- Notes: Returns index for later reference; type determines visual/behavior

### animate_scenery
- Signature: `void animate_scenery(void);`
- Purpose: Update animation frame/state for all active scenery each frame
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies scenery animation state
- Calls: Not inferable from this file
- Notes: Likely called once per game frame during update loop

### deanimate_scenery
- Signature: `void deanimate_scenery(short object_index);`
- Purpose: Stop/remove scenery animation (Lua support for runtime deletion)
- Inputs: Object index
- Outputs/Return: None
- Side effects: Deactivates scenery, may free resources
- Calls: Not inferable from this file

### damage_scenery
- Signature: `void damage_scenery(short object_index);`
- Purpose: Apply damage to a scenery object (likely triggers state change or destruction)
- Inputs: Object index
- Outputs/Return: None
- Side effects: Modifies scenery state; may activate damaged variant
- Calls: Not inferable from this file

### get_scenery_collection / get_damaged_scenery_collection
- Signature: `bool get_scenery_collection(short scenery_type, short &collection);` and damaged variant
- Purpose: Retrieve the sprite/shape collection ID for a scenery type (or damaged state variant)
- Inputs: Scenery type ID
- Outputs/Return: Boolean success; collection ID via reference
- Side effects: None
- Calls: Not inferable from this file

### parse_mml_scenery / reset_mml_scenery
- Signature: `void parse_mml_scenery(const InfoTree& root);` and `void reset_mml_scenery();`
- Purpose: Load scenery definitions from MML markup; reset to defaults
- Inputs: InfoTree (parsed XML/MML document root)
- Outputs/Return: None
- Side effects: Modifies scenery type definitions and properties
- Calls: Not inferable from this file

## Control Flow Notes
- `initialize_scenery()` runs during engine startup
- `animate_scenery()` called each frame during game update phase
- `new_scenery()` called when level loads scenery or Lua scripts spawn objects
- `parse_mml_scenery()` runs during resource/configuration loading
- `damage_scenery()` triggered by player/projectile interactions or game events

## External Dependencies
- `#include "world.h"` ΓÇö Provides `world_distance`, `world_point3d`, vector/point types
- `object_location` ΓÇö Struct defining 3D position and polygon; defined elsewhere
- `InfoTree` ΓÇö XML/MML document tree class; likely from parser module
- Primitive types: `short`, `bool`
