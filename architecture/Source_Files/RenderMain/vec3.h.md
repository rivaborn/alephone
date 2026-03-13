# Source_Files/RenderMain/vec3.h

## File Purpose
Defines lightweight vector and matrix types (vec3, vec4, vertex2, vertex3, mat4) for 3D graphics computations in the Aleph One engine. Provides basic linear algebra operations (dot product, cross product, normalization) and OpenGL integration for matrix transformations.

## Core Responsibilities
- Define homogeneous vector and vertex types for 3D graphics
- Implement vector arithmetic operators (+, ΓêÆ, ├ù, scalar multiplication)
- Provide vector math utilities (dot product, cross product, normalization)
- Define 4├ù4 matrix type with OpenGL interop
- Establish floating-point comparison tolerance for vector equality
- Support both direction vectors (w=0) and position vertices (w=1)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| vec4 | struct | 4-component homogeneous float vector; base for all vector types |
| vec3 | struct | 3-component direction vector (inherits vec4, sets w=0); core math type |
| vertex3 | struct | 3D position vertex (inherits vec4, sets w=1) |
| vertex2 | struct | 2D position vertex (inherits vec4, sets w=1) |
| mat4 | struct | 4├ù4 transformation matrix for graphics operations |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| kThreshhold | const GLfloat | file-static | Fuzzy equality tolerance for floating-point vector comparisons (FLT_MIN ├ù 10) |

## Key Functions / Methods

### vec4 (constructors and accessors)
- Signature: `vec4()`, `vec4(GLfloat x, y, z, w)`
- Purpose: Store and access 4-component vectors
- Methods: `p()` (pointer to data), `operator[]` (indexed access, const and mutable)

### vec3::operator==
- Signature: `bool operator==(const vec3& v) const`
- Purpose: Fuzzy equality comparison using Manhattan distance
- Inputs: Another vec3
- Outputs/Return: true if sum of absolute differences < kThreshhold
- Notes: Uses Manhattan (L1) distance rather than Euclidean; accounts for floating-point precision

### vec3::dot
- Signature: `GLfloat dot(const vec3& v) const`
- Purpose: Compute dot product (scalar)
- Inputs: Another vec3
- Outputs/Return: Scalar dot product value

### vec3::cross
- Signature: `vec3 cross(const vec3& v) const`
- Purpose: Compute cross product (perpendicular vector)
- Inputs: Another vec3
- Outputs/Return: New vec3 perpendicular to both inputs
- Notes: Uses right-hand rule

### vec3::length
- Signature: `GLfloat length() const`
- Purpose: Compute vector magnitude
- Outputs/Return: Euclidean norm (ΓêÜ(dot product with self))
- Calls: `dot(*this)`

### vec3::norm
- Signature: `vec3 norm() const`
- Purpose: Return unit vector (normalized direction)
- Outputs/Return: New vec3 with length Γëê 1.0
- Calls: `length()`

### vec3::copy
- Signature: `void copy(GLfloat* v)`
- Purpose: Copy vec3 components to external float array
- Inputs: Pointer to float array (assumed ΓëÑ 3 elements)
- Side effects: Writes to caller's buffer

### vertex3::operator-
- Signature: `vec3 operator-(const vertex3& v) const`
- Purpose: Compute direction vector from one vertex to another
- Inputs: Another vertex3
- Outputs/Return: vec3 (direction, w=0)

### mat4 (constructor)
- Signature: `mat4(GLenum em)`
- Purpose: Load OpenGL matrix from current state
- Inputs: OpenGL matrix mode enum (e.g., GL_PROJECTION_MATRIX)
- Side effects: Queries OpenGL state via `glGetFloatv`
- Calls: `glGetFloatv`

### mat4::operator*
- Signature: `vec4 operator*(const vec4& v) const`
- Purpose: Transform vector by matrix (matrix-vector multiplication)
- Inputs: vec4 to transform
- Outputs/Return: Transformed vec4
- **Note**: Potential bugΓÇöuses `_d[0][1]` for all matrix columns in each row; should use `_d[row][col]` pattern

### mat4::glSet
- Signature: `void glSet(GLenum em)`
- Purpose: Load matrix into OpenGL state (projection, modelview, etc.)
- Inputs: Target matrix mode (GL_PROJECTION, GL_MODELVIEW, etc.)
- Side effects: Saves/restores GL_MATRIX_MODE; calls `glLoadMatrixf`; uses push/pop attribute stack
- Calls: `glGetIntegerv`, `glPushAttrib`, `glMatrixMode`, `glLoadMatrixf`

## Control Flow Notes
No control flowΓÇöpure utility types. Used throughout rendering pipeline to represent positions, directions, and transformations passed to OpenGL. Initialized as needed during scene setup and rendering (not part of formal init/update/render phases).

## External Dependencies
- **OGL_Headers.h** ΓÇô Provides OpenGL function declarations and type definitions (GLfloat, GLenum, GL_* constants)
- **cfloat** ΓÇô Provides FLT_MIN for threshold calculation
- **cmath** ΓÇô Provides std::sqrt, std::abs for vector math
