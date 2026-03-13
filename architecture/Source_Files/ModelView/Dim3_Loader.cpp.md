# Source_Files/ModelView/Dim3_Loader.cpp

## File Purpose
Loads 3D skeletal models from Dim3 XML format for the Aleph One game engine. Parses geometry, bones, rigging, and animation data; handles multi-pass loading and bone hierarchy ordering.

## Core Responsibilities
- Parse Dim3 XML model files using InfoTree interface
- Extract and validate vertex positions, normals, and bone weights
- Build bone hierarchies and resolve bone parent-child relationships
- Read animation poses and sequences, mapping to frame names
- Convert coordinate systems and angles from Dim3 to engine conventions
- Perform topological bone sorting (depth-first traversal with push/pop flags)
- Map vertices to bone influences for skeletal animation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `BoneTagWrapper` | struct | Stores intermediate bone tag strings (e.g., bone ID and parent ID or major/minor influence) |
| `NameTagWrapper` | struct | Caches frame and animation sequence names during parsing |
| `Model3D_VertexSource` | (external) | Vertex position, bone influences (Bone0, Bone1, blend weight) |
| `Model3D_Bone` | (external) | Bone position and flags (Push/Pop for hierarchy traversal) |
| `Model3D_Frame` | (external) | Per-bone rotation and offset for a single pose frame |
| `Model3D_SeqFrame` | (external) | Animation frame reference with per-bone sway (local rotation) and offset |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `VertexBoneTags` | `vector<BoneTagWrapper>` | static | Maps vertices to major/minor bone influences; persistent across multi-file loads |
| `BoneOwnTags` | `vector<BoneTagWrapper>` | static | Maps bones to their own tag and parent tag; persistent across multi-file loads |
| `BoneIndices` | `vector<size_t>` | static | Remapping table: original bone index ΓåÆ sorted bone index; filled once and reused |
| `FrameTags` | `vector<NameTagWrapper>` | static | Frame/animation names for lookup during animation parsing |
| `Normals` | `vector<GLfloat>` | static | Temporary storage for per-vertex normals; cleared after copy to model |
| `DegreesToInternal` | const float | static | Precomputed conversion factor (512/360) for angle normalization |

## Key Functions / Methods

### GetAngle
- **Signature:** `static int16 GetAngle(float InAngle)`
- **Purpose:** Convert degrees to engine internal angle units with proper rounding and normalization.
- **Inputs:** `InAngle` (float in degrees)
- **Outputs/Return:** `int16` normalized to `[0, FULL_CIRCLE)` range
- **Side effects:** None
- **Calls:** `NORMALIZE_ANGLE` macro
- **Notes:** Handles both positive and negative inputs; negative angles rounded toward ΓêÆΓê₧.

### parse_bounding_box
- **Signature:** `static void parse_bounding_box(const InfoTree& root, Model3D& Model)`
- **Purpose:** Extract and apply bounding box dimensions and offsets from XML element.
- **Inputs:** `root` (InfoTree node), `Model` (output reference)
- **Outputs/Return:** Modifies `Model.BoundingBox[0]` and `[1]` in-place
- **Side effects:** Writes to model structure
- **Calls:** `InfoTree::read_attr()`
- **Notes:** Accepts both comma-separated "size"/"offset" attributes and individual x/y/z attributes. Default 0.5 texture coordinate if not specified.

### parse_dim3
- **Signature:** `static void parse_dim3(const InfoTree& root, Model3D& Model)`
- **Purpose:** Main XML parser; dispatches to parse vertices, bones, faces, poses, and animations.
- **Inputs:** `root` (InfoTree of `<Model>` element), `Model` (output reference)
- **Outputs/Return:** Populates `Model` and fills static vectors (`VertexBoneTags`, `BoneOwnTags`, `FrameTags`, `Normals`)
- **Side effects:** Appends to model and static global state
- **Calls:** `parse_bounding_box()`, `InfoTree::children_named()`, various `read_attr()` calls
- **Notes:** Parses in order: bounding boxes, vertices (with normals and bone tags), bones, triangle indices with texture coords, pose frames, and animation sequences. Matches animation frames to pose names via string lookup.

### LoadModel_Dim3
- **Signature:** `bool LoadModel_Dim3(FileSpecifier& Spec, Model3D& Model, int WhichPass)`
- **Purpose:** Entry point for loading Dim3 files; handles multi-pass loading, XML parsing, and post-processing.
- **Inputs:** 
  - `Spec` (file path)
  - `Model` (output structure)
  - `WhichPass` (`LoadModelDim3_First` or `LoadModelDim3_Rest`)
- **Outputs/Return:** `bool` (true if positions and indices successfully loaded)
- **Side effects:** 
  - On `WhichPass == First`: clears `Model` and all static vectors
  - Modifies `Model.Bones`, `Model.VtxSources`, normals, and animation data
  - May build `InverseVSIndices` if empty
  - Performs expensive bone topological sort on first pass
- **Calls:** `InfoTree::load_xml()`, `parse_dim3()`, `Model.BuildInverseVSIndices()`, `Model.FindPositions_Neutral()`
- **Notes:** 
  - **Bone sorting algorithm** (lines 388ΓÇô514): Performs depth-first, parent-driven traversal of bone hierarchy. Builds `BoneIndices` remapping and sets `Push`/`Pop` flags for proper skeleton traversal. Asserts on circular bone references.
  - Normal propagation copies per-vertex normals from source to indexed positions.
  - Returns false and logs error on any XML parse exception.

## Control Flow Notes
**Initialization flow:**
1. `LoadModel_Dim3()` called with `WhichPass == LoadModelDim3_First` clears model and static state
2. `InfoTree::load_xml()` loads and parses file into XML tree
3. `parse_dim3()` iterates `<Model>` elements and populates geometry, bones, faces, animations
4. Post-parse: bone hierarchy is topologically sorted; vertices are remapped to sorted bone indices
5. Normal vectors are propagated from source vertices to indexed mesh
6. Returns success/failure based on non-empty positions and indices

**Multi-file loading:** Static vectors (`BoneOwnTags`, `FrameTags`) persist across calls; subsequent passes with `WhichPass != First` append geometry without clearing.

## External Dependencies
- **Includes:** 
  - `cseries.h` (engine core types, defines)
  - `Dim3_Loader.h` (own header)
  - `world.h` (angle and coordinate definitions; `FULL_CIRCLE`, `NORMALIZE_ANGLE`)
  - `InfoTree.h` (XML parsing)
  - `Logging.h` (error logging)
- **Types from `world.h`:** `angle`, `FULL_CIRCLE` (512), `NORMALIZE_ANGLE` macro
- **Types from headers not shown:** `Model3D` (class with `VtxSources`, `Bones`, `Frames`, `SeqFrames`, bounding box, etc.), `FileSpecifier`
- **OpenGL:** Guarded by `#ifdef HAVE_OPENGL`; uses `GLfloat`, `GLushort`, `GLshort` types
