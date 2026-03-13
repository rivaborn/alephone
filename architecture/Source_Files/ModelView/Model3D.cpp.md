# Source_Files/ModelView/Model3D.cpp

## File Purpose
Implements 3D skeletal animation and vertex transformation for the Aleph One game engine. Provides core functionality for animating boned models through frame-based keyframing, interpolating between poses, and computing vertex normals for rendering. Handles coordinate transformations, bone hierarchies, and normal calculation modes (flat, smooth, split-by-threshold).

## Core Responsibilities
- **Skeletal Animation**: Compute bone matrices, traverse bone hierarchies with stack-based push/pop, blend vertex positions across dual-bone influences
- **Vertex Position Computation**: Generate final vertex positions from frame data (neutral, single-frame, or sequence-based animation)
- **Normal Processing**: Generate/normalize normals, detect hard edges by threshold, split vertices for per-polygon lighting, reverse normal directions
- **Tangent Vector Generation**: Compute tangent and bitangent vectors from texture coordinates and geometry for normal mapping
- **Transformations**: Apply point/vector transforms via 3├ù4 matrices; compose transforms hierarchically for bone chains
- **Data Management**: Clear all model data, calculate bounding boxes, build inverse index structures for efficient animation
- **Frame Interpolation**: Blend between two animation frames using angle interpolation and position blending

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Model3D_Transform | struct | 3├ù4 affine transformation matrix (rotation + translation) |
| Model3D_VertexSource | struct | Source vertex with two bone indices and blend weight |
| Model3D_Bone | struct | Bone definition: position and push/pop flags for hierarchy traversal |
| Model3D_Frame | struct | Bone pose: offset and three rotation angles (ZXY order) |
| Model3D_SeqFrame | struct | Frame with sequence metadata (extends Model3D_Frame) |
| FlaggedVector | struct (local) | 3D vector with boolean validity flag |
| Model3D | class | Container for all geometry, animation data, and transforms |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| BoneMatrices | vector\<Model3D_Transform\> | static | Temporary storage for computed bone transformation matrices during frame evaluation |
| BoneStack | vector\<size_t\> | static | Stack for traversing bone hierarchy (parent indices) |
| TrigNorm | const GLfloat | static | Normalization factor (1.0 / TRIG_MAGNITUDE) for converting trig table values to floats |

## Key Functions / Methods

### Clear()
- **Signature**: `void Model3D::Clear()`
- **Purpose**: Erase all model geometry, animation data, and metadata; reset to empty state.
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Clears all vectors (Positions, Normals, Bones, Frames, etc.); recalculates bounding box to empty state.
- **Calls**: `FindBoundingBox()`
- **Notes**: Called during model reloading or initialization.

### CalculateTangents()
- **Signature**: `void Model3D::CalculateTangents()`
- **Purpose**: Generate tangent and bitangent vectors from vertex positions and texture coordinates for normal mapping.
- **Inputs**: Populated Positions, TxtrCoords, VertIndices; optionally Normals.
- **Outputs/Return**: Populates Tangents (vec4, xyz=tangent + w=handedness sign).
- **Side effects**: Generates Normals if not present; resizes Tangents array.
- **Calls**: `VecCopy()`, vertex constructors (vertex2, vertex3), cross product, dot product, norm operations.
- **Notes**: Handles degenerate triangles by zeroing tangents; computes frame-relative tangent via cross(N,T).

### AdjustNormals(int NormalType, float SmoothThreshold)
- **Signature**: `void Model3D::AdjustNormals(int NormalType, float SmoothThreshold)`
- **Purpose**: Process normals: normalize, reverse, or split vertices by angle threshold for smooth/hard edge detection.
- **Inputs**: NormalType enum (None, Original, Reversed, ClockwiseSide, CounterclockwiseSide); SmoothThreshold variance limit.
- **Outputs/Return**: Modifies Normals; may duplicate and remap vertices.
- **Side effects**: Allocates temporary vectors (PerPolygonNormalList, PerVertexNormalList); swaps Positions, Normals, TxtrCoords, Colors, VtxSrcIndices; copies NormSources if boned model.
- **Calls**: `NormalizeNormal()`, `VecCopy()`, vector operations, cross product.
- **Notes**: ClockwiseSide/CounterclockwiseSide modes compute per-polygon normals, accumulate per-vertex, variance-split if threshold exceeded; complex memory rearrangement for split vertices.

### FindBoundingBox()
- **Signature**: `void Model3D::FindBoundingBox()`
- **Purpose**: Calculate axis-aligned bounding box from vertex positions.
- **Inputs**: Positions array (3-component floats).
- **Outputs/Return**: Populates BoundingBox[2][3] (min and max corners).
- **Side effects**: Modifies BoundingBox array.
- **Calls**: None (inline vector operations).
- **Notes**: Handles empty position list gracefully.

### BuildInverseVSIndices()
- **Signature**: `void Model3D::BuildInverseVSIndices()`
- **Purpose**: Build reverse-lookup table: for each vertex source, list all vertex indices that reference it.
- **Inputs**: VtxSrcIndices array.
- **Outputs/Return**: Populates InverseVSIndices and InvVSIPointers arrays.
- **Side effects**: Allocates two vectors; uses pointer array as temporary count storage.
- **Calls**: `objlist_clear()` (likely memset-like).
- **Notes**: Enables O(1) lookup of all vertices influenced by a bone; InvVSIPointers has one extra element.

### FindPositions_Neutral(bool UseModelTransform)
- **Signature**: `bool Model3D::FindPositions_Neutral(bool UseModelTransform)`
- **Purpose**: Copy vertex source positions into output (with optional model-wide transform).
- **Inputs**: VtxSrcIndices, VtxSources, NormSources; UseModelTransform flag; TransformPos, TransformNorm matrices.
- **Outputs/Return**: Populates Positions and Normals; returns true if vertex sources were used.
- **Side effects**: Resizes Positions and Normals vectors.
- **Calls**: `TransformPoint()`, `TransformVector()`, `objlist_copy()`.
- **Notes**: Non-animated model path; returns false if no vertex source data.

### FindPositions_Frame(bool UseModelTransform, GLshort FrameIndex, GLfloat MixFrac, GLshort AddlFrameIndex)
- **Signature**: `bool Model3D::FindPositions_Frame(bool UseModelTransform, GLshort FrameIndex, GLfloat MixFrac = 0, GLshort AddlFrameIndex = 0)`
- **Purpose**: Animate vertices for a single frame or blend two frames; handles skeletal deformation.
- **Inputs**: FrameIndex, MixFrac (interpolation weight 0ΓÇô1), AddlFrameIndex (second frame for blending); UseModelTransform flag.
- **Outputs/Return**: Populates Positions and Normals; returns false for out-of-range indices.
- **Side effects**: Resizes BoneMatrices and BoneStack; resizes Positions and Normals.
- **Calls**: `FindBoneTransform()`, `TMatMultiply()`, `TransformPoint()`, `TransformVector()`, `BuildInverseVSIndices()`, vector utilities.
- **Notes**: Core animation function; traverses bone tree via stack (Push/Pop flags); blends dual-bone vertex influences; applies cumulative parent transforms.

### FindPositions_Sequence(bool UseModelTransform, GLshort SeqIndex, GLshort FrameIndex, GLfloat MixFrac, GLshort AddlFrameIndex)
- **Signature**: `bool Model3D::FindPositions_Sequence(...)`
- **Purpose**: Animate vertices for a sequence (collection of frames with per-frame transforms).
- **Inputs**: SeqIndex (sequence ID), FrameIndex within sequence, MixFrac, AddlFrameIndex; UseModelTransform flag.
- **Outputs/Return**: Populates Positions and Normals; returns false for out-of-range indices.
- **Side effects**: Calls FindPositions_Frame(); resizes Positions, Normals.
- **Calls**: `FindFrameTransform()`, `FindPositions_Frame()`, `TMatMultiply()`, `TransformPoint()`, `TransformVector()`.
- **Notes**: Sequence adds global frame transform on top of bone animation; extracts frame from SeqFrames list.

### FindFrameTransform(Model3D_Transform& T, Model3D_Frame& Frame, GLfloat MixFrac, Model3D_Frame& AddlFrame) [static]
- **Signature**: `static void FindFrameTransform(...)`
- **Purpose**: Compute transformation matrix from frame's angles and offset (with optional interpolation).
- **Inputs**: Frame (angles ZXY and offset), MixFrac, AddlFrame (for blending).
- **Outputs/Return**: Populates T matrix.
- **Side effects**: None.
- **Calls**: `InterpolateAngle()`, cosine_table, sine_table lookups.
- **Notes**: Applies rotations in ZXY order (Tomb Raider convention); interpolates angles as shortest path.

### FindBoneTransform(Model3D_Transform& T, Model3D_Bone& Bone, Model3D_Frame& Frame, GLfloat MixFrac, Model3D_Frame& AddlFrame) [static]
- **Signature**: `static void FindBoneTransform(...)`
- **Purpose**: Compute transformation for a single bone (frame transform + bone position offset).
- **Inputs**: Bone (position), Frame (angles, offset), MixFrac, AddlFrame.
- **Outputs/Return**: Populates T matrix.
- **Side effects**: None.
- **Calls**: `FindFrameTransform()`, `ScalarProd()`.
- **Notes**: Adjusts translation to account for bone's position in overall coordinate system.

### TMatMultiply(Model3D_Transform& Res, Model3D_Transform& A, Model3D_Transform& B) [static]
- **Signature**: `static void TMatMultiply(Model3D_Transform& Res, Model3D_Transform& A, Model3D_Transform& B)`
- **Purpose**: Multiply two transformation matrices: Res = A ├ù B.
- **Inputs**: Matrices A, B.
- **Outputs/Return**: Populates Res.
- **Side effects**: None.
- **Calls**: None (arithmetic).
- **Notes**: Standard 3├ù4 matrix multiply (rotation part is 3├ù3, translation handled separately).

### TransformPoint(GLfloat *Dest, GLfloat *Src, Model3D_Transform& T) [inline]
- **Signature**: `inline void TransformPoint(GLfloat *Dest, GLfloat *Src, Model3D_Transform& T)`
- **Purpose**: Apply affine transformation to a 3D point.
- **Inputs**: Src (3 floats), T (transform).
- **Outputs/Return**: Populates Dest (3 floats).
- **Side effects**: None.
- **Calls**: `ScalarProd()`.
- **Notes**: Includes translation (T.M[i][3]); source and dest must be different arrays.

### TransformVector(GLfloat *Dest, GLfloat *Src, Model3D_Transform& T) [inline]
- **Signature**: `inline void TransformVector(GLfloat *Dest, GLfloat *Src, Model3D_Transform& T)`
- **Purpose**: Apply rotation-only transformation to a vector (no translation).
- **Inputs**: Src (3 floats), T (transform).
- **Outputs/Return**: Populates Dest (3 floats).
- **Side effects**: None.
- **Calls**: `ScalarProd()`.
- **Notes**: Omits translation component; used for normals.

### NormalizeNormal(GLfloat *Normal) [static]
- **Signature**: `static bool NormalizeNormal(GLfloat *Normal)`
- **Purpose**: Unit-normalize a 3D normal vector in-place.
- **Inputs**: Normal (3 floats).
- **Outputs/Return**: Returns true if normal had nonzero length.
- **Side effects**: Modifies Normal array.
- **Calls**: sqrt.
- **Notes**: Returns false for zero/near-zero vectors (used to flag invalid normals).

## Control Flow Notes
**Initialization**: Constructor calls `FindBoundingBox()` and initializes TransformPos/TransformNorm to identity.

**Frame Animation Flow**:
1. `FindPositions_Frame()` called with frame and optional second frame for interpolation
2. Bone matrices computed via `FindBoneTransform()` (frame angles + bone position)
3. Bone hierarchy traversed: stack Push/Pop flags compose cumulative transforms
4. For each vertex source: position blended from up to 2 bones, normals transformed
5. If UseModelTransform: final positions/normals transformed by model-wide matrix
6. Results written to output Positions/Normals arrays

**Sequence Animation**: Wraps frame animation; adds global frame-level transform via `FindFrameTransform()`.

**Normal Processing**: Varies by modeΓÇöOriginal normalizes only; ClockwiseSide/CounterclockwiseSide compute per-polygon, accumulate per-vertex, threshold-split if variance high, generating new vertices.

## External Dependencies
- **Model3D.h**: Class definition, data structure declarations, enums for normal types
- **VecOps.h**: Template vector operations (VecCopy, VecAdd, VecSub, VecScalarMult, ScalarProd, VectorProd)
- **world.h**: Trig tables (cosine_table, sine_table), angle macros (NORMALIZE_ANGLE, HALF_CIRCLE, FULL_CIRCLE), trigonometric constants (TRIG_MAGNITUDE, TRIG_SHIFT)
- **OGL_Headers.h**: OpenGL type definitions (GLfloat, GLushort, GLshort, GLubyte)
- **OGL_Setup.h**: Sgl* color macros (SglColor3fv, etc.)
- **cseries.h**: Utility macros and memory operations (objlist_clear, objlist_copy, obj_copy, MIN, MAX, TEST_FLAG, NONE, UNONE)
- **Standard C++**: \<math.h\>, \<string.h\>, \<iostream\>, \<vector\>
