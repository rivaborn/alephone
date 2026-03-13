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

## Key Types / Data Structures
| Name | Kind | Purpose |
|---|---|---|
| ModelRenderShader | struct | Render pass configuration; holds flags and function pointers for texture and lighting callbacks |
| IndexedCentroidDepth | struct | Pairs triangle index with centroid depth for depth-sorting (farthest-to-nearest) |
| ModelRenderer | class | Main renderer; maintains ViewDirection and persistent sorting/lighting data caches |

## Global / File-Static State
None.

## Key Functions / Methods

### ModelRenderer::Render
- Signature: `void Render(Model3D& Model, ModelRenderShader *Shaders, int NumShaders, int NumSeparableShaders, bool Use_Z_Buffer)`
- Purpose: Render a 3D model using one or more shader passes with optional Z-buffering and polygon depth-sorting
- Inputs:
  - `Model`: the 3D model to render
  - `Shaders`: array of shader configurations (texture/lighting callbacks + flags)
  - `NumShaders`: total shaders in array
  - `NumSeparableShaders`: count of shaders renderable separately with Z-buffer (always first); remainder are non-separable (semitransparent)
  - `Use_Z_Buffer`: true = Z-buffer depth testing; false = depth-sort polygons back-to-front
- Outputs/Return: None (side-effect rendering)
- Side effects: Invokes texture and lighting callbacks; modifies GL state; populates internal sorting caches
- Calls: `SetupRenderPass()` (once per active shader)
- Notes: Without Z-buffer, all shaders are depth-sorted together. Semitransparent shaders never render separately.

### ModelRenderer::Clear
- Signature: `void Clear()`
- Purpose: Reset persistent rendering arrays to empty state
- Inputs/Outputs: None
- Side effects: Clears IndexedCentroidDepths, SortedVertIndices, ExtLightColors vectors
- Notes: Call between render operations when reusing the renderer instance

## Control Flow Notes
ModelRenderer is a frame-scoped utility in the rendering pipeline. Typical usage: set `ViewDirection[3]`, call `Render()` with model and shader array, then optionally `Clear()`. When Z-buffer is present, separable shaders render directly; non-separable shaders are depth-sorted. Without Z-buffer, all shaders are depth-sorted back-to-front. SetupRenderPass() invokes texture and lighting callbacks to customize each pass.

## Notes
- **Trivial members**: ModelRenderShader constructor zeroes via `obj_clear()`. IndexedCentroidDepth::operator< sorts descending (farthest first). SetupRenderPass() is a private helper.
- **Render flags**: Textured (0x0001), Colored (0x0002), ExtLight (0x0004), EL_SemiTpt (0x0008).

## External Dependencies
- `csmacros.h`: `obj_clear<T>()` template for struct zero-initialization
- `Model3D.h`: `Model3D` class (mesh: vertices, indices, normals, textures, bones, animation frames)
- OpenGL types: GLfloat, GLushort (via transitively-included OGL_Headers.h)
- Standard library: `std::vector<T>`
