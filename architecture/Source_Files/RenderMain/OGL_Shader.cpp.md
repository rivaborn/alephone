# Source_Files/RenderMain/OGL_Shader.cpp

## File Purpose
Implements OpenGL shader compilation and lifecycle management for the Aleph One game engine. Provides functionality to load, compile, link, and manage GLSL vertex/fragment shader programs, including fallback error shaders and support for conditional compilation based on hardware capabilities (sRGB, bloom, etc.).

## Core Responsibilities
- Compile GLSL vertex and fragment shaders from source strings with platform-specific preprocessor directives
- Create and manage OpenGL shader programs (creation, linking, cleanup)
- Maintain a global registry of compiled shader programs indexed by type
- Set uniform variables (floats, matrices, textures) in active shaders with caching optimization
- Parse shader definitions from MML/XML configuration files and dynamically reload them
- Provide fallback error shaders when compilation fails
- Support built-in shader sources and load shader sources from disk files

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| Shader | class | Manages a single compiled OpenGL shader program; stores vertex/fragment sources, program handle, uniform locations, and cached values |
| Shader_MML_Parser | class | Static helper for parsing shader configuration from InfoTree (MML/XML) format and updating the global shader registry |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| defaultVertexPrograms | std::map<std::string, std::string> | file-static | Maps shader names to built-in GLSL vertex shader source code (populated on first call) |
| defaultFragmentPrograms | std::map<std::string, std::string> | file-static | Maps shader names to built-in GLSL fragment shader source code (populated on first call) |
| Shader::_shaders | std::vector<Shader> | class-static | Registry of all compiled shader programs, indexed by ShaderType enum |
| Shader::_uniform_names | const char* [NUMBER_OF_UNIFORM_LOCATIONS] | class-static (const) | Maps UniformName enum values to GLSL uniform variable names |
| Shader::_shader_names | const char* [NUMBER_OF_SHADER_TYPES] | class-static (const) | Maps ShaderType enum values to human-readable shader names |

## Key Functions / Methods

### parseFile
- **Signature:** `void parseFile(FileSpecifier& fileSpec, std::string& s)`
- **Purpose:** Load shader source code from disk file into a string
- **Inputs:** `fileSpec` ΓÇô file path; `s` ΓÇô output string
- **Outputs/Return:** void (result via output parameter)
- **Side effects:** File I/O; allocates memory in string; writes to stderr on failure
- **Calls:** `fileSpec.Exists()`, `fileSpec.Open()`, `file.GetLength()`, `file.Read()`
- **Notes:** Clears target string first; silently returns if file does not exist; designed to fail gracefully

### parseShader
- **Signature:** `GLhandleARB parseShader(const GLcharARB* str, GLenum shaderType)`
- **Purpose:** Compile a GLSL shader source string into a compiled GPU shader object
- **Inputs:** `str` ΓÇô shader source code; `shaderType` ΓÇô GL_VERTEX_SHADER_ARB or GL_FRAGMENT_SHADER_ARB
- **Outputs/Return:** GLhandleARB handle to compiled shader; 0 on failure
- **Side effects:** OpenGL state (creates shader, compiles); GPU memory allocation; logs compilation errors via logError()
- **Calls:** `glCreateShaderObjectARB()`, `glShaderSourceARB()`, `glCompileShaderARB()`, `glGetObjectParameterivARB()`, `glGetShaderiv()`, `glGetShaderInfoLog()`, `glDeleteObjectARB()`, `DisableClipVertex()`, `logError()`
- **Notes:** Prepends conditional #define directives based on global flags (DISABLE_CLIP_VERTEX, GAMMA_CORRECTED_BLENDING, BLOOM_SRGB_FRAMEBUFFER); error messages dumped to log on compile failure

### Shader::init
- **Signature:** `void Shader::init()`
- **Purpose:** Compile vertex and fragment shaders and link them into a complete GPU program
- **Inputs:** none (uses `_vert` and `_frag` member variables)
- **Outputs/Return:** void
- **Side effects:** Modifies `_loaded=true`, `_programObj`, `_uniform_locations[]`, `_cached_floats[]`; OpenGL state changes; GPU memory allocation; logs linking errors
- **Calls:** `parseShader()`, `glCreateProgramObjectARB()`, `glAttachObjectARB()`, `glDeleteObjectARB()`, `glLinkProgramARB()`, `glGetProgramiv()`, `glGetProgramInfoLog()`, `glUseProgramObjectARB()`, `glUniform1iARB()`, `getUniformLocation()`, `std::fill_n()`
- **Notes:** Falls back to built-in error shader if compilation fails; initializes all uniform location cache entries to -1 and float cache to 0.0; always binds texture units 0ΓÇô3 to samplers; asserts if program object creation fails

### Shader::loadAll
- **Signature:** `static void Shader::loadAll()`
- **Purpose:** Create and initialize all default shader programs (once per session)
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** Populates `_shaders` vector with NUMBER_OF_SHADER_TYPES shaders; calls `initDefaultPrograms()`
- **Calls:** `initDefaultPrograms()`, `Shader(std::string)` constructor
- **Notes:** Early exit if `_shaders` is already populated; reserves capacity before appending; defers actual GPU compilation to first enable() call

### Shader::unloadAll
- **Signature:** `static void Shader::unloadAll()`
- **Purpose:** Release all loaded shader programs and free GPU memory
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** Calls `unload()` on each shader; releases all GPU resources
- **Calls:** `unload()` for each element of `_shaders`
- **Notes:** Safe to call even if shaders not fully initialized

### Shader::Shader (name constructor)
- **Signature:** `Shader::Shader(const std::string& name)`
- **Purpose:** Create shader instance initialized with default source code by name (no compilation yet)
- **Inputs:** `name` ΓÇô shader type name (e.g., "blur", "bloom")
- **Outputs/Return:** new Shader instance
- **Side effects:** Initializes `_vert` and `_frag` from defaultVertexPrograms/defaultFragmentPrograms maps
- **Calls:** `initDefaultPrograms()`
- **Notes:** Lazy initialization; actual GPU compilation deferred to `init()` call; `_programObj=0`, `_passes=-1`, `_loaded=false`

### Shader::Shader (file constructor)
- **Signature:** `Shader::Shader(const std::string& name, FileSpecifier& vert, FileSpecifier& frag, int16& passes)`
- **Purpose:** Create shader instance from vertex and fragment file paths
- **Inputs:** `name` ΓÇô shader name; `vert`, `frag` ΓÇô file specifications; `passes` ΓÇô render pass count
- **Outputs/Return:** new Shader instance
- **Side effects:** Loads shader source from files immediately; falls back to defaults if files empty
- **Calls:** `parseFile()`, `initDefaultPrograms()`
- **Notes:** File I/O happens in constructor; GPU compilation deferred

### Shader::enable
- **Signature:** `void Shader::enable()`
- **Purpose:** Activate this shader for subsequent rendering
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** Calls `init()` if not yet loaded; activates GPU shader program via `glUseProgramObjectARB()`
- **Calls:** `init()`, `glUseProgramObjectARB()`
- **Notes:** Lazy initialization pattern; safe to call before manual `init()`

### Shader::disable
- **Signature:** `static void Shader::disable()`
- **Purpose:** Deactivate all shaders (revert to fixed-function pipeline)
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** OpenGL state change
- **Calls:** `glUseProgramObjectARB(0)`
- **Notes:** Static method; affects global GPU state

### Shader::setFloat
- **Signature:** `void Shader::setFloat(UniformName name, float f)`
- **Purpose:** Set a uniform float value in the active shader
- **Inputs:** `name` ΓÇô UniformName enum; `f` ΓÇô new value
- **Outputs/Return:** void
- **Side effects:** Updates `_cached_floats[name]`; only sends to GPU if value changed
- **Calls:** `getUniformLocation()`, `glUniform1fARB()`
- **Notes:** Caching avoids redundant GPU calls; shader must be enabled first

### Shader::setMatrix4
- **Signature:** `void Shader::setMatrix4(UniformName name, float *f)`
- **Purpose:** Set a 4├ù4 matrix uniform value in the active shader
- **Inputs:** `name` ΓÇô UniformName enum; `f` ΓÇô pointer to 16 floats (column-major)
- **Outputs/Return:** void
- **Side effects:** OpenGL state change
- **Calls:** `getUniformLocation()`, `glUniformMatrix4fvARB()`
- **Notes:** No caching (unlike setFloat); shader must be enabled first

### initDefaultPrograms
- **Signature:** `void initDefaultPrograms()`
- **Purpose:** Populate defaultVertexPrograms and defaultFragmentPrograms with all built-in shader source code (one-time initialization)
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** Fills two static maps with GLSL source strings; can only run once (early exit if already initialized)
- **Calls:** `#include` directives for external `.vert` and `.frag` files; uses preprocessor string concatenation
- **Notes:** Early exit if maps already populated; defines fallback error, gamma, blur, bloom, landscape, sprite, invincible, invisible, wall, bump, and landscape_sphere variants; texture inclusion via `#include` supports both literal strings and file-based shader source

### Shader_MML_Parser::parse
- **Signature:** `void Shader_MML_Parser::parse(const InfoTree& root)`
- **Purpose:** Parse shader definition from MML configuration and update the global shader registry
- **Inputs:** `root` ΓÇô InfoTree XML node with "name", "vert", "frag", "passes" attributes/children
- **Outputs/Return:** void
- **Side effects:** May populate `_shaders` vector if empty; modifies `_shaders[i]` for matching shader type
- **Calls:** `root.read_attr()`, `root.read_path()`, `initDefaultPrograms()`, `Shader::loadAll()`, `Shader()` constructor
- **Notes:** Matches "name" attribute against _shader_names array to find index; silently returns if name not found

### Shader_MML_Parser::reset
- **Signature:** `static void Shader_MML_Parser::reset()`
- **Purpose:** Clear shader registry for configuration reload
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** Clears `Shader::_shaders` vector (does not call unload on individual shaders)
- **Calls:** `_shaders.clear()`
- **Notes:** Called by `reset_mml_opengl_shader()`

## Control Flow Notes
**Initialization phase:** `OGL_Initialize()` ΓåÆ `Shader::loadAll()` creates all shader instances; `initDefaultPrograms()` populates source maps; GPU compilation deferred.

**Per-frame rendering:** Caller invokes `shader.enable()` ΓåÆ shader calls `init()` if needed ΓåÆ caller invokes `setFloat()`/`setMatrix4()` to update frame-varying uniforms ΓåÆ drawing commands execute ΓåÆ caller may invoke `Shader::disable()`.

**Configuration reload:** MML parser calls `reset_mml_opengl_shader()` ΓåÆ `Shader_MML_Parser::reset()` ΓåÆ `parse_mml_opengl_shader()` ΓåÆ `Shader_MML_Parser::parse()` updates individual shader sources and re-initializes them on next `enable()`.

**Shutdown:** `OGL_Shutdown()` ΓåÆ `Shader::unloadAll()` releases all GPU resources.

## External Dependencies
- **STL:** `<algorithm>` (std::fill_n), `<iostream>` (fprintf), `<string>`, `<map>`
- **File I/O:** FileHandler.h (FileSpecifier, OpenedFile)
- **OpenGL ARB:** glCreateShaderObjectARB, glShaderSourceARB, glCompileShaderARB, glCreateProgramObjectARB, glAttachObjectARB, glLinkProgramARB, glUseProgramObjectARB, glGetObjectParameterivARB, glGetShaderiv, glGetProgramiv, glGetUniformLocationARB, glGetShaderInfoLog, glGetProgramInfoLog, glDeleteObjectARB, glDeleteProgram, glUniform1iARB, glUniform1fARB, glUniformMatrix4fvARB
- **OGL Configuration:** OGL_Setup.h (global flags: Wanting_sRGB, Bloom_sRGB, DisableClipVertex())
- **Configuration parsing:** InfoTree.h (XML/MML tree abstraction)
- **Logging:** Logging.h (logError macro)
