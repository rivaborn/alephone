# Source_Files/RenderMain/scottish_textures.h

## File Purpose
Defines core texture rendering structures and transfer modes for the Aleph One engine's software and OpenGL renderers. Provides intermediate representations (sprites, wall polygons, floor/ceiling) that bridge game logic and rendering subsystems, with support for tinting, shading, lighting, and 3D model rendering.

## Core Responsibilities
- Define transfer modes for texture rendering (tinted, solid, textured, static, shadeless, landscaped)
- Provide tinting/shading table structures for 8-bit, 16-bit, and 32-bit color rendering
- Define `rectangle_definition` struct for sprites and floor/ceiling textures with position, depth, clipping, and opacity
- Define `polygon_definition` struct for wall polygons with vertices, origin, and shading
- Support OpenGL 3D model rendering via shape descriptors and model pointers
- Manage ambient and ceiling lighting for both software and OpenGL paths
- Track world-space positioning and projected depth for proper lighting and sorting

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| tint_table8 | struct | 8-bit palette-based color tinting lookup table |
| tint_table16 | struct | 16-bit RGB component tinting tables |
| tint_table32 | struct | 32-bit RGB component tinting tables |
| point2d | struct | 2D integer screen coordinates |
| rectangle_definition | struct | Sprite/floor/ceiling texture instance with rendering state, 3D position, model data, and lighting |
| polygon_definition | struct | Wall polygon with screen vertices, 3D origin, shading tables, and rendering parameters |
| OGL_ModelData | class | Forward declaration; opaque container for OpenGL model rendering state |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| bit_depth | short | extern | Current rendering bit depth (8, 16, or 32 bpp) |
| interface_bit_depth | short | extern | UI rendering bit depth |
| number_of_shading_tables | short | extern | Count of active shading tables |
| shading_table_fractional_bits | short | extern | Fractional precision bits in shade indices |
| shading_table_size | short | extern | Memory size of each shading table |

## Key Functions / Methods

### allocate_texture_tables
- Signature: `void allocate_texture_tables(void)`
- Purpose: Initialize and allocate memory for global shading/tinting tables based on bit depth
- Inputs: None (uses global `bit_depth`, `number_of_shading_tables`)
- Outputs/Return: None (void); modifies global shading table state
- Side effects: Allocates dynamic memory for shading tables; called during engine initialization
- Calls: Not visible in header (defined in `SCOTTISH_TEXTURES.C`)
- Notes: Must be called before any rendering occurs; size depends on color depth and table configuration

## Constants & Enums

**Transfer Modes** (6 types):
- `_tinted_transfer`: Applies indexed shading table; supports tint masks
- `_solid_transfer`: Fills non-transparent pixels with texture (0,0) color
- `_big_landscaped_transfer`: Anchors texture in screen-space (no perspective distortion)
- `_textured_transfer`: Standard perspective-correct texturing
- `_shadeless_transfer`: Ignores lighting; uses single shading table
- `_static_transfer`: Fills pixels with noise based on probability config

**Vertex Limits**: 3ΓÇô16 vertices per screen polygon

**Render Flags**:
- `_SHADELESS_BIT` (0x8000): Ignore per-polygon shading tables
- `_SCALE_2X_BIT`, `_SCALE_4X_BIT`: Floor/ceiling 2├ù and 4├ù scaling

## Control Flow Notes
Structures are populated during the render frame:
1. Game logic updates world geometry and sprite positions
2. Renderer projects polygons and sprites to screen space, populating `rectangle_definition` and `polygon_definition`
3. Renderer queries world lighting and computes ambient/ceiling shade
4. For OpenGL: attaches shape descriptors and model data pointers; for software: references shading tables
5. Rendering subsystem (rasterizer or OpenGL) consumes these structures and performs rasterization

The `depth` and `ProjDistance` fields enable depth-based lighting; the `LightDepth` field supports "miner's light" (attenuation by distance to light source).

## External Dependencies
- **cseries.h**: Basic types (`pixel8`, `pixel16`, `pixel32`, `_fixed`), platform macros
- **OGL_Headers.h**: OpenGL/GLEW headers
- **world.h**: World geometry (`world_point3d`, `world_vector3d`, `long_point3d`)
- **shape_descriptors.h**: Shape descriptor type and collection/shape/clut macros
- **OGL_ModelData**: Opaque class (forward-declared; defined elsewhere) for 3D model state
- **Defined elsewhere**: `allocate_texture_tables()` implementation in `SCOTTISH_TEXTURES.C`
