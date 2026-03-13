# Source_Files/RenderOther/OverheadMap_OGL.cpp

## File Purpose
OpenGL implementation of overhead map (automap) rendering for Marathon game engine. Provides hardware-accelerated 2D map display with cached geometry batching for performance optimization.

## Core Responsibilities
- Clear screen and manage OpenGL state for 2D overhead map rendering
- Implement polygon rendering with color-change batching (triangle fans)
- Implement line rendering with width and color caching
- Render game world objects (things) as circles or rectangles at specified sizes
- Render player position and orientation as triangles with proper transformation
- Render map annotations (text) with font support and justification
- Render monster/npc paths as line strips
- Optimize rendering via geometry caching (flush on parameter changes)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `rgb_color` | struct | Color definition (red, green, blue channels) |
| `world_point2d` | struct | 2D world coordinates |
| `FontSpecifier` | class | Font specification for text rendering (defined elsewhere) |
| `angle` | typedef | Angle type for player facing direction |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ViewWidth` | short | extern | Viewport width (from OGL_Render.cpp) |
| `ViewHeight` | short | extern | Viewport height (from OGL_Render.cpp) |
| `SavedColor` | rgb_color | member | Current color for batch rendering |
| `SavedPenSize` | short | member | Current line width for batch rendering |
| `PolygonCache` | vector<unsigned short> | member | Cached polygon vertex indices |
| `LineCache` | vector<world_point2d> | member | Cached line endpoints |
| `PathPoints` | vector<world_point2d> | member | Cached path points for line strips |

## Key Functions / Methods

### begin_overall()
- **Purpose**: Initialize OpenGL state for overhead map rendering frame
- **Inputs**: None
- **Outputs/Return**: void
- **Side effects**: Blanks screen with black polygon; disables depth test, alpha test, fog, texture 2D; enables blend if translucent map
- **Calls**: `OGL_RenderRect()`, `map_is_translucent()`, OpenGL state functions
- **Notes**: Called once per frame before drawing map geometry

### end_overall()
- **Purpose**: Restore OpenGL texture coordinate state after overhead map rendering
- **Inputs**: None
- **Outputs/Return**: void
- **Side effects**: Re-enables `GL_TEXTURE_COORD_ARRAY`
- **Calls**: OpenGL state functions
- **Notes**: Pairs with `begin_overall()`

### begin_polygons() / end_polygons()
- **Purpose**: Set up polygon rendering batch and flush cached polygons
- **Inputs**: None
- **Outputs/Return**: void
- **Side effects**: `begin_polygons()` sets vertex pointer, initializes SavedColor, clears PolygonCache; `end_polygons()` flushes cache
- **Calls**: `glVertexPointer()`, `SetColor()`, `DrawCachedPolygons()`
- **Notes**: Polygons rendered as triangle fans via GL_TRIANGLES

### draw_polygon()
- **Purpose**: Add polygon to render cache, flushing if color changes
- **Inputs**: `vertex_count` (short), `vertices` (short*), `color` (rgb_color&)
- **Outputs/Return**: void
- **Side effects**: Modifies PolygonCache; may call `DrawCachedPolygons()` and `SetColor()`
- **Calls**: `ColorsEqual()`, `DrawCachedPolygons()`, `SetColor()`
- **Notes**: Converts polygon to triangle fan by creating triangles from first vertex and consecutive pairs; batch flushes on color change

### DrawCachedPolygons()
- **Purpose**: Render all cached polygon triangles via OpenGL
- **Inputs**: None
- **Outputs/Return**: void
- **Side effects**: Clears PolygonCache after rendering
- **Calls**: `glDrawElements(GL_TRIANGLES, ...)`
- **Notes**: Uses vertex pointer set in `begin_polygons()`

### begin_lines() / end_lines()
- **Purpose**: Initialize line rendering and flush cached lines
- **Inputs**: None
- **Outputs/Return**: void
- **Side effects**: `begin_lines()` sets default color/pen size, clears LineCache; `end_lines()` flushes cache
- **Calls**: `SetColor()`, `DrawCachedLines()`

### draw_line()
- **Purpose**: Add line to cache, flushing on color or width change
- **Inputs**: `vertices` (short*), `color` (rgb_color&), `pen_size` (short)
- **Outputs/Return**: void
- **Side effects**: Modifies LineCache, SavedColor, SavedPenSize; may call `DrawCachedLines()`
- **Calls**: `ColorsEqual()`, `GetVertex()`, `DrawCachedLines()`, `SetColor()`
- **Notes**: Batch flushes on color or pen-size change

### DrawCachedLines()
- **Purpose**: Render all cached lines with current pen width
- **Inputs**: None
- **Outputs/Return**: void
- **Side effects**: Clears LineCache after rendering
- **Calls**: `OGL_RenderLines()` (defined elsewhere)

### draw_thing()
- **Purpose**: Render a map object (circle or rectangle) at specified location and size
- **Inputs**: `center` (world_point2d&), `color` (rgb_color&), `shape` (short), `radius` (short)
- **Outputs/Return**: void
- **Side effects**: Modifies GL_MODELVIEW matrix; calls OpenGL state/rendering functions
- **Calls**: `SetColor()`, `glMatrixMode()`, `glPushMatrix()`, `glTranslatef()`, `glScalef()`, `OGL_RenderRect()`, `glVertexPointer()`, `glDrawArrays()`, `glPopMatrix()`
- **Notes**: Switches on `shape` (_rectangle_thing, _circle_thing); circle uses 18 vertices in triangle strip; uses matrix transformations for position/scale

### draw_player()
- **Purpose**: Render player position, facing direction, and shape as triangle on map
- **Inputs**: `center` (world_point2d&), `facing` (angle), `color` (rgb_color&), `shrink` (short), `front` (short), `rear` (short), `rear_theta` (short)
- **Outputs/Return**: void
- **Side effects**: Modifies GL_MODELVIEW matrix; disables textures and texture arrays
- **Calls**: `SetColor()`, `glMatrixMode()`, `glPushMatrix()`, `glTranslatef()`, `glRotatef()`, `glScalef()`, `glDisable()`, `glDisableClientState()`, `glVertexPointer()`, `glDrawArrays()`, `glPopMatrix()`
- **Notes**: Constructs 3-vertex triangle with front/rear/width; applies rotation for facing direction; scales based on shrink factor

### draw_text()
- **Purpose**: Render text annotation on overhead map with justification
- **Inputs**: `location` (world_point2d&), `color` (rgb_color&), `text` (char*), `FontData` (FontSpecifier&), `justify` (short)
- **Outputs/Return**: void
- **Side effects**: Modifies GL_MODELVIEW matrix; sets font near-filter from HUD texture info
- **Calls**: `SetColor()`, `glMatrixMode()`, `glPushMatrix()`, `glLoadIdentity()`, `glTranslatef()`, `FontData.OGL_Render()`, `glPopMatrix()`
- **Notes**: Supports left and center justification; uses FontSpecifier to render glyphs (font handling is macOS-specific per comments)

### set_path_drawing()
- **Purpose**: Set color for subsequent path drawing
- **Inputs**: `color` (rgb_color&)
- **Outputs/Return**: void
- **Side effects**: Calls `SetColor()`
- **Calls**: `SetColor()`

### draw_path() / finish_path()
- **Purpose**: Accumulate path points and render as continuous line strip
- **Inputs**: `step` (short), `location` (world_point2d&)
- **Outputs/Return**: void
- **Side effects**: Modifies PathPoints; `finish_path()` clears it after rendering
- **Calls**: `PathPoints.push_back()`, `OGL_RenderLines()`
- **Notes**: `step <= 0` resets path; duplicates final point to form lines; path rendered as single line strip for efficiency

## Control Flow Notes
**Frame lifecycle:**
1. `begin_overall()` - Clear screen, disable depth/alpha/fog, enable blend if translucent
2. Polygons: `begin_polygons()` ΓåÆ multiple `draw_polygon()` ΓåÆ `end_polygons()`
3. Lines: `begin_lines()` ΓåÆ multiple `draw_line()` ΓåÆ `end_lines()`
4. Objects/player/text rendered directly between
5. `end_overall()` - Restore texture state

Caching is crucial: polygons/lines flush only on parameter (color/width) changes, enabling batch rendering of same-property geometry.

## External Dependencies
- **OpenGL**: `glColor3usv()`, `glColor4us()`, `glMatrixMode()`, `glPushMatrix()`, `glPopMatrix()`, `glTranslatef()`, `glRotatef()`, `glScalef()`, `glVertexPointer()`, `glDrawElements()`, `glDrawArrays()`, state queries/changes
- **OGL_Render.h**: `OGL_RenderRect()`, `OGL_RenderLines()`, extern `ViewWidth`/`ViewHeight`
- **OGL_Textures.h**: `TxtrTypeInfoList[]` (for HUD texture filter info)
- **OverheadMapRenderer.h**: Base class `OverheadMapClass` (parent implementation not provided)
- **map.h**: `rgb_color`, `world_point2d`, `world_point3d`, `angle` type definitions
- **screen.h**: `map_is_translucent()` function
- **FontSpecifier**: Font rendering interface (defined elsewhere)
