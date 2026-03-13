# Subsystem Overview

## Purpose
The ModelView subsystem loads, stores, and renders 3D skeletal models in multiple file formats (Dim3 XML, 3D Studio Max, Wavefront OBJ), providing unified internal representation, skeletal animation playback with frame interpolation, and OpenGL rendering with multi-pass shader support.

## Key Files
| File | Role |
|------|------|
| Model3D.h | Defines Model3D data structures: vertex geometry, skeletal animation, bones, frames, sequences, bounding boxes, and tangent vectors. |
| Model3D.cpp | Implements skeletal animation (bone matrices, vertex blending, frame interpolation), normal generation, tangent computation, coordinate transformations, and vertex position calculations. |
| Dim3_Loader.h / Dim3_Loader.cpp | Loads 3D skeletal models from Dim3 XML format; parses geometry, bones, rigging, and animation data with topological bone sorting. |
| StudioLoader.h / StudioLoader.cpp | Loads 3D Studio Max .3ds files; parses hierarchical chunk format and extracts vertex positions, texture coordinates, and polygon indices. |
| WavefrontLoader.h / WavefrontLoader.cpp | Loads Wavefront OBJ files; parses geometry, handles vertex deduplication, fan tessellation, and coordinate system conversion. |
| ModelRenderer.h | Defines ModelRenderer class for OpenGL rendering with multi-pass shaders, depth-sorting, and texture/lighting callbacks. |
| ModelRenderer.cpp | Implements OpenGL rendering with Z-buffer or centroid-based depth-sorting; batches separable shaders to reduce draw calls. |

## Core Responsibilities
- Load 3D models from Dim3 XML, 3D Studio Max (.3ds), and Wavefront OBJ file formats
- Parse geometry (vertex positions, normals, texture coordinates), skeletal data (bones, hierarchy, vertex weights), and animation sequences
- Compute skeletal animation via bone matrix transforms, vertex blending across dual-bone influences, and frame interpolation with angle blending
- Generate and normalize vertex normals; support flat, smooth, and split-by-threshold modes for hard-edge detection
- Compute tangent and bitangent vectors for normal mapping
- Perform coordinate system conversions (right-handed to left-handed) across loaders
- Render 3D models to OpenGL with multi-pass shader support, texture binding, and external lighting
- Implement centroid-based depth-sorting for polygon rendering when Z-buffer is unavailable
- Optimize rendering by batching separable shaders and caching sorted indices and lighting colors

## Key Interfaces & Data Flow
- **Exposes:** LoadModel_Dim3, LoadModel_Studio, LoadModel_Wavefront (public loader functions); Model3D class with animation methods; ModelRenderer class for drawing
- **Consumes:** FileSpecifier (FileHandler), InfoTree (XML parsing), world.h (trig tables, angle macros FULL_CIRCLE, NORMALIZE_ANGLE), cseries.h (platform macros, memory operations), OGL_Headers.h (OpenGL types)
- **Data Flow:** File loaders populate Model3D structures from disk; Model3D exposes animation frame computation; ModelRenderer draws Model3D geometry via OpenGL

## Runtime Role
- Loaders execute during map/resource initialization to populate Model3D structures from files
- Model3D animation methods are called during frame updates to compute current vertex positions and normals
- ModelRenderer is invoked during render passes to draw model geometry with current animation state and shader configuration

## Notable Implementation Details
- Skeletal animation uses stack-based push/pop for efficient depth-first bone hierarchy traversal and matrix composition
- Vertex blending supports dual-bone influences with weighted position averaging
- Bone hierarchy is topologically sorted (depth-first) to ensure parent matrices are computed before children
- Normal calculation modes detect hard edges via threshold comparison and split vertices for per-polygon lighting
- ModelRenderer optimizes draw call count by identifying and batching separable shaders into single multi-pass calls
- All loaders handle coordinate system conversion; geometry may convert from right-handed (standard 3D formats) to left-handed (engine convention)
- Centroid-based depth-sorting caches polygon centroids and sorted triangle indices to amortize computation across frames
