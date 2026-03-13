# Source_Files/ModelView/WavefrontLoader.h

## File Purpose
This is a loader module for Wavefront OBJ 3D model files in the Aleph One game engine. It provides two entry points: one for direct loading without coordinate system conversion, and one that automatically converts from OBJ's right-handed to Aleph One's left-handed coordinate system.

## Core Responsibilities
- Load Wavefront OBJ model files via FileSpecifier abstraction
- Parse OBJ geometry data into Model3D object structure
- Optionally transform vertex and texture coordinates between coordinate systems
- Provide flexible loading options for different use cases
- Integrate with engine's model storage and file I/O systems

## Key Types / Data Structures
None

## Global / File-Static State
None

## Key Functions / Methods

### LoadModel_Wavefront
- Signature: `bool LoadModel_Wavefront(FileSpecifier& Spec, Model3D& Model)`
- Purpose: Load a Wavefront OBJ model without any coordinate system transformation
- Inputs: `Spec` (file specification/path), `Model` (destination model structure)
- Outputs/Return: `bool` (success/failure)
- Side effects: Populates the `Model` parameter with parsed geometry data (vertices, normals, texture coordinates, faces)
- Calls: (not visible; implementation in .cpp file)
- Notes: Direct loading preserving OBJ's native right-handed coordinates; caller responsible for any further conversions

### LoadModel_Wavefront_RightHand
- Signature: `bool LoadModel_Wavefront_RightHand(FileSpecifier& Spec, Model3D& Model)`
- Purpose: Load a Wavefront OBJ model and convert its vertex and texture coordinates from OBJ's right-handed to Aleph One's left-handed coordinate system
- Inputs: `Spec` (file specification/path), `Model` (destination model structure)
- Outputs/Return: `bool` (success/failure)
- Side effects: Populates the `Model` parameter with parsed and transformed geometry data
- Calls: (not visible; implementation in .cpp file)
- Notes: Automatically applies handedness conversion during loading; more convenient than manual conversion for typical OBJ assets

## Control Flow Notes
Not inferable from this header file alone. These are model asset loading entry points, likely invoked during scene setup or asset loading phases. Actual parsing and transformation logic resides in the implementation (.cpp) file.

## External Dependencies
- `Model3D.h`: Defines Model3D struct containing geometry storage (positions, normals, texture coordinates, vertex indices, bones, animation frames)
- `FileHandler.h`: Provides FileSpecifier class for abstract cross-platform file path handling
- Standard library: `<stdio.h>` (for FILE-related types if used in implementation)
