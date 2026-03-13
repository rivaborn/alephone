# Source_Files/ModelView/Dim3_Loader.h

## File Purpose
Header for the Dim3 3D model format loader. Provides an interface to load 3D geometry, skeleton, and animation data from Dim3 model files into the engine's Model3D representation, supporting multifile models via a two-pass loading mechanism.

## Core Responsibilities
- Declare the main model loading function `LoadModel_Dim3`
- Define enumeration for controlling multifile model loading passes
- Abstract away Dim3 file format specifics from callers

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| (anonymous enum) | enum | Pass control for multipass model loading |

## Global / File-Static State
None.

## Key Functions / Methods

### LoadModel_Dim3
- **Signature:** `bool LoadModel_Dim3(FileSpecifier& Spec, Model3D& Model, int WhichPass);`
- **Purpose:** Load a Dim3 format model file into a Model3D object, supporting multiple file passes.
- **Inputs:**
  - `Spec`: File specification pointing to the Dim3 model file
  - `Model`: Reference to Model3D object to populate (vertices, bones, frames, sequences, etc.)
  - `WhichPass`: `LoadModelDim3_First` for initial setup, `LoadModelDim3_Rest` for subsequent files
- **Outputs/Return:** Boolean indicating success/failure
- **Side effects:** Modifies the Model3D object's vertex, bone, frame, and sequence data; performs file I/O
- **Calls:** Defined elsewhere (implementation in .cpp file)
- **Notes:** Multipass design allows combining geometry from multiple Dim3 files into a single Model3D. First pass typically initializes state; subsequent passes accumulate data.

## Control Flow Notes
Part of the model loading pipeline. Called during asset initialization to populate Model3D structures used later for rendering and skeletal animation. The pass parameter enables combining multiple Dim3 files (common for complex models with separate geometry, skeleton, and animation files).

## External Dependencies
- **Model3D.h:** `Model3D` struct ΓÇö stores vertex positions, texture coordinates, normals, bones, frames, sequences, and transforms
- **FileHandler.h:** `FileSpecifier` class ΓÇö file abstraction for cross-platform file access
