# Source_Files/GameWorld/physics_models.h

## File Purpose
Defines the physics models and constants for character movement (walking and running) in the game engine. Provides data structures to parameterize velocity, acceleration, angular movement, and collision geometry for player characters. Declares serialization functions to pack/unpack physics constants from binary data.

## Core Responsibilities
- Define enumeration of physics model types (`_model_game_walking`, `_model_game_running`)
- Declare `physics_constants` struct containing all physics parameters (velocities, accelerations, angular properties, dimensions)
- Provide hardcoded default physics constants for two movement models
- Declare serialization/deserialization functions for physics data (binary I/O)
- Declare initialization function for physics constants at engine startup

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `physics_constants` | struct | Encapsulates all physics parameters for a movement model (linear/angular velocities, accelerations, actor dimensions, camera offset) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `original_physics_models` | const array of `physics_constants[NUMBER_OF_PHYSICS_MODELS]` | global | Hardcoded default physics constants for walking and running models, indexed by model enum |

## Key Functions / Methods

### unpack_physics_constants
- Signature: `uint8 *unpack_physics_constants(uint8 *Stream, physics_constants *Objects, size_t Count)`
- Purpose: Deserialize physics constant structures from binary stream
- Inputs: byte stream pointer, destination objects array, count of objects to unpack
- Outputs/Return: advanced stream pointer (post-read position)
- Side effects: modifies memory at Objects pointer; advances stream position
- Calls: (not inferable from this file)
- Notes: stream-based deserialization suggests file I/O or network transmission of physics data

### pack_physics_constants
- Signature: `uint8 *pack_physics_constants(uint8 *Stream, physics_constants *Objects, size_t Count)`
- Purpose: Serialize physics constant structures to binary stream
- Inputs: byte stream pointer, objects array to serialize, count of objects
- Outputs/Return: advanced stream pointer (post-write position)
- Side effects: writes to stream; advances stream position
- Calls: (not inferable from this file)
- Notes: mirror of unpack function; enables persistence/network transmission

### unpack_m1_physics_constants
- Signature: `uint8* unpack_m1_physics_constants(uint8* Stream, size_t Count)`
- Purpose: Deserialize Marathon 1 (legacy) format physics constants from stream
- Inputs: byte stream pointer, count of objects
- Outputs/Return: advanced stream pointer
- Side effects: (not inferable from this file)
- Calls: (not inferable from this file)
- Notes: compatibility function for loading old game data format

### init_physics_constants
- Signature: `void init_physics_constants()`
- Purpose: Initialize physics constants at engine startup
- Inputs: none
- Outputs/Return: none
- Side effects: likely populates runtime physics data structures (not inferable from header)
- Calls: (not inferable from this file)
- Notes: called during engine initialization phase

## Control Flow Notes
Physics constants are loaded/initialized at engine startup via `init_physics_constants()`. The `original_physics_models` array serves as hardcoded defaults indexed by model type. Serialization functions support loading physics from disk files (likely .cfg or binary game data). Physics parameters govern character movement behavior during runtime update/frame logic.

## External Dependencies
- **Includes:** `world.h` ΓÇö provides fixed-point arithmetic (`_fixed` type), angle constants (`QUARTER_CIRCLE`), distance types (`world_distance`)
- **External symbols:** `FIXED_ONE`, `QUARTER_CIRCLE` macros from world.h; `_fixed`, `uint8` types from world.h/cstypes.h
- **Implicit:** Standard C integer types (`size_t`, `uint8`)
