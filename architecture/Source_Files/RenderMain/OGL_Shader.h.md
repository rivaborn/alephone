# Source_Files/RenderMain/OGL_Shader.h

## File Purpose
Defines the `Shader` class for managing OpenGL vertex/fragment shader compilation, linking, and runtime state. Provides static factory methods to access pre-loaded shader instances and uniform variable binding for rendering pipeline integration.

## Core Responsibilities
- Compile and link OpenGL shader programs from vertex/fragment source files
- Manage uniform variable locations and cached values for per-frame updates
- Support multiple shader types (landscape, sprite, wall, bloom effects, infravision, etc.)
- Provide global shader instance access and lifecycle management (load all / unload all)
- Enable/disable shaders and set uniform float and matrix values during rendering
- Support multi-pass rendering through `passes()` member

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `UniformName` | enum | Identifies all shader uniforms (textures, time, effects like pulsate/wobble/flare/bloom, camera transforms, depth modes, gamma, etc.) |
| `ShaderType` | enum | Identifies all shader variants (blur, landscape, sprite, wall, bump, with bloom and infravision variants) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_shader_names` | `const char*[NUMBER_OF_SHADER_TYPES]` | static | Maps `ShaderType` enum values to string identifiers |
| `_shaders` | `std::vector<Shader>` | static | Global repository of all compiled shader instances |
| `_uniform_names` | `const char*[NUMBER_OF_UNIFORM_LOCATIONS]` | static | Maps `UniformName` enum values to GLSL uniform variable names |
| `_uniform_locations` | `GLint[NUMBER_OF_UNIFORM_LOCATIONS]` | member | Cached OpenGL uniform location handles (lazy-initialized per instance) |
| `_cached_floats` | `float[NUMBER_OF_UNIFORM_LOCATIONS]` | member | Cached float uniform values to optimize redundant calls |

## Key Functions / Methods

### `static Shader* get(ShaderType type)`
- **Purpose:** Factory method to retrieve a pre-loaded shader instance by type
- **Inputs:** `ShaderType` enum value
- **Outputs/Return:** Pointer to shader in `_shaders` vector
- **Side effects:** None (read-only access)
- **Calls:** Array indexing on `_shaders`

### `static void loadAll()` / `static void unloadAll()`
- **Purpose:** Global initialization/cleanup of all shader instances; typically called at engine startup and shutdown
- **Inputs:** None
- **Side effects:** Loads/unloads all entries in `_shaders` vector
- **Calls:** Individual `load()` / `unload()` per shader

### `load()` / `init()`
- **Purpose:** `load()` reads shader source from disk; `init()` compiles and links the program
- **Inputs:** Shader name and file paths (set via constructor)
- **Outputs/Return:** None; sets `_loaded` flag and `_programObj`
- **Side effects:** OpenGL compilation, GPU memory allocation
- **Calls:** OpenGL ARB shader functions (not visible in header)

### `enable()`
- **Purpose:** Activate shader for subsequent rendering calls
- **Side effects:** Sets current OpenGL shader program
- **Calls:** OpenGL shader activation function

### `static void disable()`
- **Purpose:** Deactivate the current shader program
- **Side effects:** Clears active OpenGL shader

### `setFloat(UniformName name, float value)` / `setMatrix4(UniformName name, float *f)`
- **Purpose:** Update uniform variable on GPU; shader must be enabled
- **Inputs:** Uniform enum identifier and float or matrix data
- **Side effects:** GPU state update; caches float values in `_cached_floats`
- **Calls:** `getUniformLocation()`, OpenGL `glUniform*` functions

### `getUniformLocation(UniformName name)` (private)
- **Purpose:** Lazy-load and cache OpenGL uniform location handle
- **Side effects:** Caches location in `_uniform_locations[]`

### `int16 passes()`
- **Purpose:** Return number of rendering passes required for this shader
- **Outputs/Return:** `_passes` member value

## Control Flow Notes
Fits into **initialization** and **per-frame rendering**:
- **Init phase:** `loadAll()` ΓåÆ `init()` compiles all shaders at startup
- **Frame phase:** Game loop calls `enable()` on appropriate shader, then `setFloat()`/`setMatrix4()` to bind per-frame state (camera, time, effects)
- **Shutdown:** `unloadAll()` ΓåÆ `unload()` releases GPU resources

Shader selection is likely driven by object type (landscape vs. sprite) and active effects (bloom, infravision).

## External Dependencies
- **OGL_Headers.h** ΓÇö `GLhandleARB`, OpenGL function declarations (glGetUniformLocationARB, etc.)
- **FileHandler.h** ΓÇö `FileSpecifier` class for file I/O
- **std::string, std::map** ΓÇö Standard library
- **Friend classes:** `XML_ShaderParser`, `Shader_MML_Parser` (defined elsewhere; parsers for shader definitions)
- **Global functions:** `parse_mml_opengl_shader(const InfoTree&)`, `reset_mml_opengl_shader()` ΓÇö MML-based shader configuration (defined elsewhere; `InfoTree` is forward-declared)
