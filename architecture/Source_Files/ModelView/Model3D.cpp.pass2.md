# Source_Files/ModelView/Model3D.cpp - Enhanced Analysis

## Architectural Role

Model3D is the **animation-to-geometry bridge** for skeletal models, sitting between GameWorld (which owns animation frame indices) and the rendering pipeline (which requires transformed vertex data). It is the only place in the engine where bone hierarchies are evaluated and vertex positions are computed from animation frames. Every animated 3D model in the game passes through `FindPositions_*()` methods each frame before rendering. This file is idiomatic to the **Tomb Raider-era** game architecture (1990s skeletal animation design) and handles the sophisticated vertex-splitting logic for hard-edge lighting that was necessary before modern deferred rendering.

## Key Cross-References

### Incoming (who depends on this file)

- **OGL_Model_Def.cpp** ΓÇö Loads model definitions (geometry, bones, frames, sequences) and stores Model3D objects
- **RenderMain (render.cpp, RenderPlaceObjs.h)** ΓÇö Calls `FindPositions_Frame()` / `FindPositions_Sequence()` before rendering each animated model instance
- **Files/game_wad.cpp** ΓÇö Loads model binary data from WAD archives and populates Model3D members
- **OGL_Render.cpp** ΓÇö Submits transformed vertex/normal data to OpenGL after animation

### Outgoing (what this file depends on)

- **world.h** ΓÇö Critically depends on `cosine_table[]`, `sine_table[]`, angle macros (`NORMALIZE_ANGLE`, `HALF_CIRCLE`), and `TRIG_MAGNITUDE` constant for angle interpolation and rotation matrix construction
- **VecOps.h** ΓÇö Template vector operations for position blending, cross/dot products, normalization
- **OGL_Headers.h / OGL_Setup.h** ΓÇö GLfloat, color macros (only for type definitions; actual rendering is elsewhere)

## Design Patterns & Rationale

### 1. **Stack-Based Bone Hierarchy Traversal** (Tomb Raider pattern)
The `BoneStack` vector simulates a call stack for parent-to-child bone transforms. Each bone has `Push` and `Pop` flags that control whether its transform is accumulated or reset. This avoids recursive function calls (fast, cache-friendly) and maps elegantly to the WAD binary format where bones are stored in linear order with push/pop metadata.

**Why this design**: Allows arbitrary branching bone hierarchies (e.g., torso ΓåÆ left/right arms, left/right legs) without pointer indirection per bone. The linear storage in WAD format directly feeds the stack iteration.

### 2. **Lazy Temporary Allocation** (pre-STL era optimization)
`BoneMatrices` and `BoneStack` are static vectors that resize on each frame evaluation. This avoids per-frame heap allocation in the hot path but trades memory safety for performance. Modern engines would use arena allocators or pre-sized buffers.

**Tradeoff**: Speed (reuse allocations) vs. clarity (unclear ownership and size expectations).

### 3. **Dual-Bone Vertex Blending** (skeletal soft-skinning)
Each vertex can be influenced by up to 2 bones with a blend weight. `Model3D_VertexSource` stores two bone indices and a scalar blend factor. This is more flexible than rigid vertex-to-bone assignment but simpler than full linear-blend skinning (LBS). Computing both bone transforms and interpolating is O(1) per vertex.

**Why**: Smooth deformation at joints; reasonable CPU cost in the 1990s era.

### 4. **Vertex Splitting for Hard Edges** (pre-normal-map era lighting)
`AdjustNormals()` with `ClockwiseSide`/`CounterclockwiseSide` mode duplicates vertices along sharp creases (variance threshold). This allows per-polygon normals to coexist with smooth normals, enabling hard edges without explicit tangent data. The logic:
- Compute per-polygon normals (via cross product)
- Accumulate per-vertex (smooth-shaded default)
- If variance exceeds threshold, duplicate vertex once per adjacent polygon
- Remap triangle indices to new vertices

**Why**: Pre-dates tangent-space normal mapping. Fake hard edges via vertex duplication is memory-intensive but renders correctly without additional data per-vertex.

## Data Flow Through This File

### Animation Evaluation (per frame, per model):

```
Game Frame (30 FPS)
    Γåô
[RenderMain] calls FindPositions_Frame(frameIndex, mixFrac, addlFrameIndex)
    Γåô
Model3D::FindPositions_Frame()
    Γö£ΓöÇ Validate frame indices
    Γö£ΓöÇ Resize BoneMatrices, BoneStack
    Γö£ΓöÇ FOR each vertex source:
    Γöé  Γö£ΓöÇ BuildInverseVSIndices() ΓåÆ map vertex-source ΓåÆ affected vertices
    Γöé  Γö£ΓöÇ FindBoneTransform() for primary bone (frame angles + offset)
    Γöé  Γö£ΓöÇ Optional: FindBoneTransform() for secondary bone (weight blend)
    Γöé  Γö£ΓöÇ Traverse bone hierarchy: push/pop transforms ΓåÆ cumulative matrix
    Γöé  Γö£ΓöÇ TransformPoint(vertex_pos) ΓåÆ world space
    Γöé  ΓööΓöÇ TransformVector(normal) ΓåÆ rotated normal
    Γö£ΓöÇ Optional: apply model-wide transform (TransformPos, TransformNorm)
    ΓööΓöÇ Return Positions[], Normals[] arrays to caller
        Γåô
[RenderMain] passes arrays to OpenGL for rendering
```

### Normal Processing Path:

```
CalculateTangents() [optional, one-time]:
    Γö£ΓöÇ Generate Normals if absent
    Γö£ΓöÇ FOR each triangle:
    Γöé  Γö£ΓöÇ Compute tangent from texture coord derivatives
    Γöé  Γö£ΓöÇ Compute bitangent from normal ├ù tangent
    Γöé  ΓööΓöÇ Store tangent[4] = (T.x, T.y, T.z, handedness)
    ΓööΓöÇ Return Tangents[] for GPU normal mapping

AdjustNormals(NormalType, SmoothThreshold):
    Γö£ΓöÇ Copy NormSources ΓåÆ Normals (if boned model)
    Γö£ΓöÇ IF ClockwiseSide/CounterclockwiseSide:
    Γöé  Γö£ΓöÇ Compute per-polygon normals (cross products)
    Γöé  Γö£ΓöÇ Accumulate per-vertex
    Γöé  Γö£ΓöÇ Variance test: split vertices above threshold
    Γöé  Γö£ΓöÇ Remap vertex indices to new duped vertices
    Γöé  ΓööΓöÇ Update all position, normal, texture coord arrays
    ΓööΓöÇ Else: just normalize vectors
```

## Learning Notes

### What This File Teaches About the Aleph One Architecture

1. **Bone Hierarchies Were Pre-Computed** ΓÇö Unlike modern skinning (GPU-evaluated), bones and transforms are traversed **per-model-instance** on the CPU. This is fine for 30 FPS fixed-step games but would be slow for thousands of concurrent animated models.

2. **Frame Interpolation is Manual** ΓÇö Angle blending uses `InterpolateAngle()` (shortest-path slerp-like logic) rather than quaternions. Frame offsets are linearly interpolated. This is era-appropriate but less numerically stable than modern quaternion SLERP.

3. **Vertex Source Indirection** ΓÇö Models store vertex data *once* in `VtxSources[]`, and vertices reference them by index. This saves memory (multiple vertices can share a source) but requires an indirection loop. BuildInverseVSIndices() is the spatial index making this tolerable.

4. **No GPU Skinning** ΓÇö Vertex transformation is entirely CPU-side. Bone matrices are uploaded or baked into each model instance, not computed by shaders. This was necessary for DirectX 5 / early OpenGL targets.

5. **Idiomatic Tomb Raider Pattern** ΓÇö ZXY rotation order, dual-bone blending, and the binary WAD format all point to this being a Marathon/Aleph One design inspired by (or descended from) Tomb Raider's engine circa 1996ΓÇô2001.

### Modern Equivalents

- **Dual-bone blending** ΓåÆ Modern **Linear Blend Skinning (LBS)** with arbitrary bone count per vertex
- **Stack-based hierarchy** ΓåÆ GPU-computed **bone matrix palettes** in shaders
- **Vertex splitting** ΓåÆ **Normal maps** or deferred rendering
- **Manual angle interpolation** ΓåÆ **Quaternion SLERP** or dual quaternions for rotation

## Potential Issues

1. **Numeric Instability in Angle Blending** ΓÇö `InterpolateAngle()` uses shortest-path logic but may have edge cases near multiples of `HALF_CIRCLE`. No quaternion normalization or stability guarantees.

2. **AdjustNormals Complexity** ΓÇö The vertex-splitting logic in `ClockwiseSide` mode is dense and difficult to verify. A single indexing error in `IndicesInList` or `VertexPolygons` remapping could silently corrupt vertex data. No assertions validate remapping correctness.

3. **Static Vector Resizing in Hot Path** ΓÇö `BoneMatrices.resize()` and `BoneStack.resize()` on every frame can trigger reallocation if model complexity varies. No pre-sizing or pool strategy. Modern frame pacing could expose this.

4. **Trig Table Dependency** ΓÇö Requires `world.h` trig tables to be initialized and `TRIG_MAGNITUDE` to match the table's encoding. Silent overflow/underflow if misaligned. No validation.

5. **No Bounds Checking on Frame/Bone Indices** ΓÇö `FindBoneTransform()` does not validate that bone indices in frames are within bounds. OOB access would read uninitialized matrix data.

6. **Boned vs. Non-Boned Models** ΓÇö `NormSources` copy path suggests models can have pre-baked normals that are transformed, but this split code path is not obvious and could be a latent bug if normals are updated asynchronously.
