# Source_Files/RenderMain/Rasterizer_SW.h

## File Purpose
Software-based rasterizer implementation for the game engine's rendering pipeline. Provides concrete texture rendering methods for horizontal/vertical polygons and rectangles via the Scottish Textures system. Inherits from the abstract `RasterizerClass` to support multiple rendering backends.

## Core Responsibilities
- Implement software-based polygon and rectangle rasterization
- Manage view/camera data and screen framebuffer pointers
- Provide entry points for the Scottish Textures rendering system
- Support both horizontal and vertical polygon orientations
- Maintain compatibility with the abstract rasterizer interface

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Rasterizer_SW_Class` | class | Concrete software rasterizer implementation inheriting from `RasterizerClass` |
| `view_data` | (external struct) | Camera/view transformation and projection data |
| `bitmap_definition` | (external struct) | Output framebuffer representation |
| `polygon_definition` | (external struct) | Textured polygon geometry and attributes |
| `rectangle_definition` | (external struct) | Textured rectangle geometry and attributes |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `view` | `view_data*` | member | Points to active view/camera data for current frame |
| `screen` | `bitmap_definition*` | member | Points to output framebuffer; named for scottish_textures compatibility |

## Key Functions / Methods

### SetView
- Signature: `void SetView(view_data& View)`
- Purpose: Configure the active camera/view data before rendering
- Inputs: Reference to `view_data` struct
- Outputs/Return: None
- Side effects: Updates member pointer `view`
- Calls: None (setter)
- Notes: Must be called before any rendering operations; inherited virtual from base class

### texture_horizontal_polygon
- Signature: `void texture_horizontal_polygon(polygon_definition& textured_polygon)`
- Purpose: Rasterize a horizontally-oriented textured polygon
- Inputs: Reference to `polygon_definition` (texture, vertices, etc.)
- Outputs/Return: Pixels written to `screen` framebuffer
- Side effects: Modifies screen framebuffer memory
- Calls: Implementation in scottish_textures.c (not visible here)
- Notes: Inherited virtual from base class; implementation location documented in comment

### texture_vertical_polygon
- Signature: `void texture_vertical_polygon(polygon_definition& textured_polygon)`
- Purpose: Rasterize a vertically-oriented textured polygon
- Inputs: Reference to `polygon_definition`
- Outputs/Return: Pixels written to `screen` framebuffer
- Side effects: Modifies screen framebuffer memory
- Calls: Implementation in scottish_textures.c
- Notes: Separate method for vertical orientation (likely optimization/specialization)

### texture_rectangle
- Signature: `void texture_rectangle(rectangle_definition& textured_rectangle)`
- Purpose: Rasterize a textured rectangle (axis-aligned primitive)
- Inputs: Reference to `rectangle_definition`
- Outputs/Return: Pixels written to `screen` framebuffer
- Side effects: Modifies screen framebuffer memory
- Calls: Implementation in scottish_textures.c
- Notes: Simpler primitive than polygons; may use optimized code path

## Control Flow Notes
Part of the rendering backend abstraction layer. Initialization: `SetView()` called once per frame or camera change. Rendering: During each frame, the three texture methods are called for different primitive types encountered in the scene graph. The actual rasterization logic is outsourced to scottish_textures.c, suggesting this header defines the interface contract while implementation varies.

## External Dependencies
- **Base class**: `RasterizerClass` (Rasterizer.h) ΓÇö defines virtual interface for swappable rasterizer backends
- **Type definitions** (defined elsewhere): `view_data`, `bitmap_definition`, `polygon_definition`, `rectangle_definition` ΓÇö likely in render.h or related headers
- **Implementation**: scottish_textures.c ΓÇö contains actual rasterization algorithms (not visible in this header)
