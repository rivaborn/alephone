# Source_Files/ModelView/StudioLoader.h

## File Purpose
Interface for loading 3D Studio MAX model files into the Aleph One game engine. Provides two loading functions: one that preserves the model's native right-handed coordinate system, and one that converts it to the engine's left-handed system.

## Core Responsibilities
- Declare model loader functions for 3DS Max format files
- Abstract file I/O through `FileSpecifier` reference
- Populate `Model3D` structures with loaded geometry
- Handle coordinate system transformation (right-handed ΓåÆ left-handed)

## Key Types / Data Structures
None defined in this file (forward references only).

| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | class | Defined in `FileHandler.h`; abstracts file path and I/O operations |
| `Model3D` | struct | Defined in `Model3D.h`; container for 3D mesh data (vertices, normals, indices, bones, frames) |

## Global / File-Static State
None.

## Key Functions / Methods

### LoadModel_Studio
- **Signature:** `bool LoadModel_Studio(FileSpecifier& Spec, Model3D& Model);`
- **Purpose:** Load a 3D Studio MAX model file without modifying its coordinate system.
- **Inputs:** `Spec` ΓÇö file specification (path to .3ds file); `Model` ΓÇö empty Model3D to populate
- **Outputs/Return:** `bool` ΓÇö success/failure; populated `Model` (output parameter)
- **Side effects:** Reads file I/O; modifies `Model` in place
- **Calls:** Not inferable from this file (implementation in .cpp)
- **Notes:** Preserves the native right-handed coordinate system of 3DS Max files

### LoadModel_Studio_RightHand
- **Signature:** `bool LoadModel_Studio_RightHand(FileSpecifier& Spec, Model3D& Model);`
- **Purpose:** Load a 3D Studio MAX model file and convert vertex/texture coordinates from right-handed to left-handed (Aleph One's system).
- **Inputs:** `Spec` ΓÇö file specification; `Model` ΓÇö empty Model3D to populate
- **Outputs/Return:** `bool` ΓÇö success/failure; populated and coordinate-converted `Model`
- **Side effects:** Reads file I/O; modifies `Model` in place with coordinate transforms
- **Calls:** Not inferable from this file (implementation in .cpp)
- **Notes:** Both vertex positions and texture coordinates are transformed

## Control Flow Notes
This header is part of the asset-loading pipeline, likely invoked during level/model initialization or streaming. The implementation (.cpp) would parse the binary 3DS format, populate vertex arrays, indices, normals, and optionally skeletal/animation data from `Model3D`, and handle format-specific coordinate conversion if the `_RightHand` variant is called.

## External Dependencies
- `<stdio.h>` ΓÇö C standard I/O (likely for file utilities)
- `Model3D.h` ΓÇö 3D mesh and skeletal animation data structures
- `FileHandler.h` ΓÇö `FileSpecifier` class for cross-platform file abstraction
