# Source_Files/ModelView/Model3D.h

## File Purpose
Defines the OpenGL-friendly data structures and interfaces for storing 3D model geometry, skeletal animation, and rendering metadata. Supports both static geometry and animated models with bone-based vertex deformation and frame/sequence animations.

## Core Responsibilities
- Store vertex geometry (positions, normals, texture coordinates, colors, tangents)
- Manage skeletal animation infrastructure (bones, frames, sequences, vertex blending)
- Compute and manipulate normals (original, reversed, per-face, smoothed)
- Calculate tangent vectors for normal mapping
- Track and transform model bounding boxes
- Provide animation interpolation via frame crossfading
- Support inverse vertex-source lookup for efficient skeletal deformation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Model3D_VertexSource` | struct | Source vertex with two-bone blending weights; used as input for skeletal animation |
| `Model3D_Bone` | struct | Bone definition with position, stack-based tree traversal flags (Push/Pop) |
| `Model3D_Frame` | struct | Per-frame bone transform (offset + 3-axis angles in lookup-table form) |
| `Model3D_SeqFrame` | struct | Extends Model3D_Frame; adds frame index for sequence management |
| `Model3D_Transform` | struct | 3├ù4 transformation matrix for model-space to render-space conversion |
| `Model3D` | struct | Master model container; holds all geometry, animation, and metadata |

## Global / File-Static State
None.

## Key Functions / Methods

### Constructor: `Model3D()`
- **Purpose**: Initialize a new model with identity transforms and compute initial bounding box.
- **Inputs**: None.
- **Outputs/Return**: New Model3D instance.
- **Side effects**: Calls `FindBoundingBox()` and `Identity()` on transform matrices.
- **Notes**: Always initializes transforms; bounding box may be empty if no positions yet added.

### `FindBoundingBox()`
- **Purpose**: Compute axis-aligned bounding box from current position data.
- **Inputs**: (implicit) Positions array.
- **Outputs/Return**: Stores result in `BoundingBox[2][3]` (min/max corners).
- **Side effects**: Modifies BoundingBox member.

### `AdjustNormals(int NormalType, float SmoothThreshold)`
- **Purpose**: Recalculate or transform normals according to type (Original, Reversed, ClockwiseSide, CounterclockwiseSide, None) and optionally smooth across shared vertices.
- **Inputs**: NormalType enum, smoothing threshold (default 0.5).
- **Outputs/Return**: None; modifies Normals array.
- **Side effects**: Regenerates normal data; may split vertices if not averaging.
- **Notes**: None/Original both use AdjustNormals internally; NormalizeNormals is convenience wrapper.

### `NormalizeNormals()`
- **Purpose**: Renormalize all normals to unit length.
- **Inputs**: None.
- **Outputs/Return**: None.
- **Side effects**: Modifies Normals array.
- **Calls**: Invokes `AdjustNormals(Original)`.

### `CalculateTangents()`
- **Purpose**: Compute per-vertex tangent vectors (4D: direction + handedness) for normal-mapped rendering.
- **Inputs**: (implicit) Positions, normals, texture coordinates.
- **Outputs/Return**: Populates Tangents array.
- **Side effects**: Modifies Tangents vector.

### `Clear()`
- **Purpose**: Erase all model data.
- **Inputs**: None.
- **Outputs/Return**: None.
- **Side effects**: Clears all vertex/bone/frame/animation arrays.

### `BuildTrigTables()` (static)
- **Purpose**: Pre-build trigonometric lookup tables for angle-based transforms (Marathon engine convention).
- **Inputs**: None.
- **Outputs/Return**: None.
- **Side effects**: Global state (if not already initialized elsewhere).
- **Notes**: Only needed if world.h's `build_trig_tables()` was not called.

### `BuildInverseVSIndices()`
- **Purpose**: Construct reverse lookup from vertex-sources to vertex indices; used to accelerate vertex-source deformation.
- **Inputs**: (implicit) VtxSrcIndices, VtxSources.
- **Outputs/Return**: Populates InverseVSIndices and InvVSIPointers.
- **Side effects**: Regenerates inverse index arrays.

### `FindPositions_Neutral(bool UseModelTransform)`
- **Purpose**: Calculate vertex positions in neutral (non-animated) state, optionally applying model transform.
- **Inputs**: UseModelTransform flag.
- **Outputs/Return**: bool (whether vertex-source data was used).
- **Side effects**: Modifies Positions array.
- **Notes**: Used for rigged models in rest pose.

### `FindPositions_Frame(bool UseModelTransform, GLshort FrameIndex, GLfloat MixFrac, GLshort AddlFrameIndex)`
- **Purpose**: Deform vertices to frame pose, with optional crossfade to a second frame.
- **Inputs**: UseModelTransform flag, frame index, blend fraction (0ΓÇô1), optional additional frame index.
- **Outputs/Return**: bool (indices in range).
- **Side effects**: Modifies Positions array.
- **Notes**: Out-of-range indices silently fall back to neutral. MixFrac=0 ΓåÆ FrameIndex only; 1 ΓåÆ AddlFrameIndex only.

### `FindPositions_Sequence(bool UseModelTransform, GLshort SeqIndex, GLshort FrameIndex, GLfloat MixFrac, GLshort AddlFrameIndex)`
- **Purpose**: Deform vertices using sequence animation with frame crossfading.
- **Inputs**: UseModelTransform flag, sequence index, frame index within sequence, blend fraction, optional additional frame.
- **Outputs/Return**: bool (indices in range).
- **Side effects**: Modifies Positions array.
- **Calls**: Internally uses frame-lookup via SeqFrmPointers.

### `NumSeqFrames(GLshort SeqIndex)`
- **Purpose**: Query frame count in a specific sequence.
- **Inputs**: Sequence index.
- **Outputs/Return**: GLshort frame count (0 if out of range).

### Helper accessors (trivial)
- `PosBase()`, `TCBase()`, `NormBase()`, `TangentBase()`, `ColBase()`, `VtxSIBase()`, `VtxSrcBase()`, `NormSrcBase()`, `InverseVIBase()`, `InvVSIPtrBase()`, `BoneBase()`, `VIBase()`, `FrameBase()`, `SeqFrmBase()`, `SFPtrBase()` ΓÇö return base pointers for OpenGL vertex arrays.
- `NumVI()` ΓÇö returns triangle vertex index count.
- `TrueNumFrames()` ΓÇö returns Frames.size() / Bones.size(); 0 if no bones.
- `TrueNumSeqs()` ΓÇö returns max(SeqFrmPointers.size() - 1, 0).

## Control Flow Notes
This file defines data layout for the model/rendering pipeline. Initialization typically flows: load geometry ΓåÆ FindBoundingBox() ΓåÆ AdjustNormals() ΓåÆ CalculateTangents(). Animation uses FindPositions_Frame() or FindPositions_Sequence() during render-loop updates to compute deformed vertex positions. The dual-index scheme (vertex-sources + inverse indices) supports efficient per-bone deformation lookups.

## External Dependencies
- **OpenGL types**: GLfloat, GLshort, GLushort (via OGL_Headers.h)
- **Standard library**: `std::vector` (STL)
- **Custom math**: `vec3`, `vec4` from vec3.h
- **Platform abstractions**: cseries.h
- **Conditional compilation**: HAVE_OPENGL guard
