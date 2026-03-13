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

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### Render
- **Signature:**  
  `void Render(Model3D& Model, ModelRenderShader *Shaders, int NumShaders, int NumSeparableShaders, bool Use_Z_Buffer)`

- **Purpose:**  
  Render a 3D model with one or more shaders, depth-sorting triangles if required.

- **Inputs:**  
  - `Model`: 3D model with positions, indices, normals, colors, texture coords
  - `Shaders`: Array of shader configurations
  - `NumShaders`: Total shaders to apply
  - `NumSeparableShaders`: Count of shaders that can be rendered independently (when Z-buffer present)
  - `Use_Z_Buffer`: Whether hardware Z-buffer is available

- **Outputs/Return:**  
  None (renders to OpenGL state).

- **Side effects:**  
  Sets up OpenGL vertex arrays, texture arrays, color arrays; issues draw calls; modifies `IndexedCentroidDepths`, `SortedVertIndices` caches; clears `NumSeparableShaders` if no Z-buffer.

- **Calls:**  
  `SetupRenderPass()`, `std::sort()`, OpenGL: `glEnableClientState()`, `glVertexPointer()`, `glDrawElements()`.

- **Notes:**  
  - Early returns if no shaders or positions.
  - **Fast path:** If all shaders separable and Z-buffer present, render without depth-sort.
  - **Depth-sort path:** Compute triangle centroid depths along view direction, sort farthest-to-nearest, render separable shaders batched, then non-separable per-triangle.
  - Separable shaders are assumed to be first in the array.

### SetupRenderPass
- **Signature:**  
  `void SetupRenderPass(Model3D& Model, ModelRenderShader& Shader)`

- **Purpose:**  
  Configure OpenGL rendering state (texture, colors, lighting) for a single shader pass.

- **Inputs:**  
  - `Model`: 3D model (for textures, normals, colors, positions)
  - `Shader`: Shader with flags, texture callback, lighting callback

- **Outputs/Return:**  
  None.

- **Side effects:**  
  Enables/disables GL_TEXTURE_2D, GL_TEXTURE_COORD_ARRAY, GL_COLOR_ARRAY; sets pointers for texture coords and colors; calls `Shader.TextureCallback` and `Shader.LightingCallback`; resizes `ExtLightColors` cache.

- **Calls:**  
  `Shader.TextureCallback()`, `Shader.LightingCallback()`, OpenGL state functions.

- **Notes:**  
  - **Texture path:** Enable if model has texture coords and shader is `Textured` flag set.
  - **Lighting path:** If shader has lighting callback and model has normals, compute per-vertex lit colors; blend with model colors if `Colored` flag set.
  - **Color path:** If no external lighting, use model colors directly if present.
  - Handles 3-channel (RGB) and 4-channel (RGBA) external lighting.
  - Asserts that `Shader.TextureCallback` is non-null.

### Clear
- **Signature:**  
  `void Clear()`

- **Purpose:**  
  Release cached vertex indices and lighting color buffers.

- **Inputs:**  
  None.

- **Outputs/Return:**  
  None.

- **Side effects:**  
  Clears `IndexedCentroidDepths`, `SortedVertIndices`, `ExtLightColors` vectors.

- **Notes:**  
  Called to free memory when renderer is no longer in use.

## Control Flow Notes
- **Per-frame:** `Render()` is the entry point, called once per frame per model.
- **Shader separation:** If Z-buffer + all separable ΓåÆ single draw per shader (fast). Otherwise ΓåÆ depth-sort all triangles, batch separable shaders, then draw non-separable per-triangle (slower but correct blending order).
- **Shutdown:** `Clear()` called to release persistent buffers.

## External Dependencies
- **OpenGL:** `glEnableClientState`, `glDisableClientState`, `glVertexPointer`, `glTexCoordPointer`, `glColorPointer`, `glEnable`, `glDisable`, `glDrawElements`
- **STL:** `<algorithm>` (`std::sort`), `<string.h>`, `<stdlib.h>`
- **Headers:** `cseries.h` (platform/macro defs), `ModelRenderer.h` (class, shader structs, flags), `Model3D.h` (external; model data)
- **Defines:** `HAVE_OPENGL` (compilation guard)
