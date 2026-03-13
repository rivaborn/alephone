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

# Source_Files/ModelView/Dim3_Loader.h
## File Purpose
Header for the Dim3 3D model format loader. Provides an interface to load 3D geometry, skeleton, and animation data from Dim3 model files into the engine's Model3D representation, supporting multifile models via a two-pass loading mechanism.

## Core Responsibilities
- Declare the main model loading function `LoadModel_Dim3`
- Define enumeration for controlling multifile model loading passes
- Abstract away Dim3 file format specifics from callers

## External Dependencies
- **Model3D.h:** `Model3D` struct ΓÇö stores vertex positions, texture coordinates, normals, bones, frames, sequences, and transforms
- **FileHandler.h:** `FileSpecifier` class ΓÇö file abstraction for cross-platform file access

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

## External Dependencies
- **Model3D.h**: Class definition, data structure declarations, enums for normal types
- **VecOps.h**: Template vector operations (VecCopy, VecAdd, VecSub, VecScalarMult, ScalarProd, VectorProd)
- **world.h**: Trig tables (cosine_table, sine_table), angle macros (NORMALIZE_ANGLE, HALF_CIRCLE, FULL_CIRCLE), trigonometric constants (TRIG_MAGNITUDE, TRIG_SHIFT)
- **OGL_Headers.h**: OpenGL type definitions (GLfloat, GLushort, GLshort, GLubyte)
- **OGL_Setup.h**: Sgl* color macros (SglColor3fv, etc.)
- **cseries.h**: Utility macros and memory operations (objlist_clear, objlist_copy, obj_copy, MIN, MAX, TEST_FLAG, NONE, UNONE)
- **Standard C++**: \<math.h\>, \<string.h\>, \<iostream\>, \<vector\>

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

## External Dependencies
- **OpenGL types**: GLfloat, GLshort, GLushort (via OGL_Headers.h)
- **Standard library**: `std::vector` (STL)
- **Custom math**: `vec3`, `vec4` from vec3.h
- **Platform abstractions**: cseries.h
- **Conditional compilation**: HAVE_OPENGL guard

# Source_Files/ModelView/ModelRenderer.cpp
## File Purpose
Implements OpenGL rendering of 3D models with multi-pass shader support and depth-sorting. Handles both Z-buffer and software-sorted rendering paths, optimizing separable shader batching to reduce draw calls.

## Core Responsibilities
- Render 3D models with one or more shader passes using OpenGL
- Perform centroid-based depth-sorting when Z-buffer unavailable or non-separable shaders present
- Setup OpenGL state for texture, color, and external lighting per render pass
- Optimize separable shader rendering by batching into single draw call
- Reduce per-triangle rendering overhead for non-separable shaders by grouping separable ones
- Cache sorted indices and lighting colors to avoid per-frame reallocation

## External Dependencies
- **OpenGL:** `glEnableClientState`, `glDisableClientState`, `glVertexPointer`, `glTexCoordPointer`, `glColorPointer`, `glEnable`, `glDisable`, `glDrawElements`
- **STL:** `<algorithm>` (`std::sort`), `<string.h>`, `<stdlib.h>`
- **Headers:** `cseries.h` (platform/macro defs), `ModelRenderer.h` (class, shader structs, flags), `Model3D.h` (external; model data)
- **Defines:** `HAVE_OPENGL` (compilation guard)

# Source_Files/ModelView/ModelRenderer.h
## File Purpose
Defines the ModelRenderer class for rendering 3D models with optional Z-buffering. Supports multipass rendering via callback-based texture and lighting shaders, and performs depth-sorting of polygons when Z-buffer is unavailable.

## Core Responsibilities
- Render 3D models with configurable multi-pass shader rendering
- Perform depth-sorting of model triangles by centroid when Z-buffer is absent
- Execute separable vs. non-separable (semitransparent) shader passes
- Manage texture and lighting callbacks for per-pass customization
- Support external lighting colors with optional semitransparency
- Maintain persistent caches (IndexedCentroidDepths, SortedVertIndices, ExtLightColors) to avoid re-allocation

## External Dependencies
- `csmacros.h`: `obj_clear<T>()` template for struct zero-initialization
- `Model3D.h`: `Model3D` class (mesh: vertices, indices, normals, textures, bones, animation frames)
- OpenGL types: GLfloat, GLushort (via transitively-included OGL_Headers.h)
- Standard library: `std::vector<T>`

# Source_Files/ModelView/StudioLoader.cpp
## File Purpose
Loads 3D Studio Max (.3ds) model files for the Aleph One game engine. Parses the hierarchical chunk-based file format, extracts vertex positions, texture coordinates, and polygon indices, and optionally converts 3D coordinates from right-handed to left-handed orientation.

## Core Responsibilities
- Parse 3DS file format structure (chunk headers, container chunks, leaf data chunks)
- Recursively traverse the Master ΓåÆ Editor ΓåÆ Object ΓåÆ Trimesh ΓåÆ geometry data hierarchy
- Buffer and deserialize chunk data with little-endian byte order
- Extract vertex positions (3 floats per vertex) from VERTICES chunks
- Extract texture coordinates (2 floats per vertex) from TXTR_COORDS chunks
- Extract polygon face indices from FACE_DATA chunks
- Convert coordinate systems and winding order (right-handed to left-handed)
- Provide error logging and validation throughout file reading

## External Dependencies
- **Logging.h:** logError, logTrace, logNote macros
- **Packing.h:** StreamToValue, StreamToList template functions for little-endian binary deserialization
- **StudioLoader.h:** Public function declarations
- **Model3D.h:** Model3D struct with Positions, VertIndices, TxtrCoords vectors and accessor methods (PosBase, VIBase, TCBase)
- **FileHandler.h:** OpenedFile class (Read, SetPosition, GetPosition); FileSpecifier class (Open, GetPath)
- **cseries.h:** Platform types (uint8, uint16, uint32, int32, GLfloat); vector, string, assertions

# Source_Files/ModelView/StudioLoader.h
## File Purpose
Interface for loading 3D Studio MAX model files into the Aleph One game engine. Provides two loading functions: one that preserves the model's native right-handed coordinate system, and one that converts it to the engine's left-handed system.

## Core Responsibilities
- Declare model loader functions for 3DS Max format files
- Abstract file I/O through `FileSpecifier` reference
- Populate `Model3D` structures with loaded geometry
- Handle coordinate system transformation (right-handed ΓåÆ left-handed)

## External Dependencies
- `<stdio.h>` ΓÇö C standard I/O (likely for file utilities)
- `Model3D.h` ΓÇö 3D mesh and skeletal animation data structures
- `FileHandler.h` ΓÇö `FileSpecifier` class for cross-platform file abstraction

# Source_Files/ModelView/WavefrontLoader.cpp
## File Purpose
Loads and parses Wavefront OBJ 3D model files for the Aleph One engine. Converts OBJ geometry (vertices, texture coordinates, normals) into the engine's internal `Model3D` representation. Includes optional coordinate-system conversion from OBJ's right-handed to Aleph One's left-handed convention.

## Core Responsibilities
- Parse Wavefront OBJ file format, handling line continuations (backslash escapes)
- Extract and buffer vertex positions, texture coordinates, and surface normals
- Parse face definitions and convert vertex index sets (with per-component indices: position/texture/normal)
- Deduplicate vertex sets to build a unique vertex list and index remapping
- Tessellate arbitrary polygons into triangles via fan decomposition
- Validate index ranges and presence of required data (positions are mandatory)
- Convert geometry and normals from right-handed (OBJ convention) to left-handed (engine convention)
- Provide detailed error, warning, and trace logging throughout parsing

## External Dependencies
- **Framework:** `Model3D`, `FileSpecifier` (engine types)
- **Logging:** `logNote`, `logError`, `logWarning`, `logTrace` macros
- **STL:** `vector`, `sort`, `algorithm`
- **Standard C/C++:** `ctype.h`, `stdlib.h`, `string.h`, `cstring`, `<algorithm>`

# Source_Files/ModelView/WavefrontLoader.h
## File Purpose
This is a loader module for Wavefront OBJ 3D model files in the Aleph One game engine. It provides two entry points: one for direct loading without coordinate system conversion, and one that automatically converts from OBJ's right-handed to Aleph One's left-handed coordinate system.

## Core Responsibilities
- Load Wavefront OBJ model files via FileSpecifier abstraction
- Parse OBJ geometry data into Model3D object structure
- Optionally transform vertex and texture coordinates between coordinate systems
- Provide flexible loading options for different use cases
- Integrate with engine's model storage and file I/O systems

## External Dependencies
- `Model3D.h`: Defines Model3D struct containing geometry storage (positions, normals, texture coordinates, vertex indices, bones, animation frames)
- `FileHandler.h`: Provides FileSpecifier class for abstract cross-platform file path handling
- Standard library: `<stdio.h>` (for FILE-related types if used in implementation)


