# Source_Files/GameWorld/world.h

## File Purpose
Defines foundational world coordinate systems, geometric types, and mathematical utilities for the Aleph One game engine. Provides fixed-point and integer-based coordinate representations (world_distance vs. long int32), trigonometric helpers, and spatial transformation functions for 2D/3D geometry operations.

## Core Responsibilities
- Define world coordinate distance types and fractional-bit constants (WORLD_FRACTIONAL_BITS = 10)
- Provide dual coordinate hierarchies: `world_*` (int16-based, lower precision) and `long_*` (int32-based, higher precision) for 2D/3D points and vectors
- Declare trigonometric tables (sine/cosine) and angle normalization macros
- Declare spatial transformation functions (rotation, translation, composition) for both 2D and 3D points
- Declare distance calculation functions and integer square root
- Declare world location structures that bundle position, orientation, polygon index, and velocity
- Provide overflow handling for long-distance coordinates via flags-based kludges

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `angle` | typedef | 16-bit angle type, range [0, 512) representing full circle |
| `fixed_angle` | typedef | Fixed-point (16.16) precision angle |
| `world_distance` | typedef | 16-bit signed world coordinate distance |
| `long_vector2d` | struct | 2D vector with int32 components; supports `+=`, `-=`, `*=` operators |
| `long_vector3d` | struct | 3D vector with int32 components; can project to 2D via `.ij()` |
| `long_point2d` | struct | 2D point with int32 coordinates; vector arithmetic support |
| `long_point3d` | struct | 3D point with int32 coordinates; can project to 2D via `.xy()` |
| `world_vector2d` | struct | 2D vector with int16 components; implicit cast to `long_vector2d` |
| `world_vector3d` | struct | 3D vector with int16 components; implicit cast to `long_vector3d` |
| `world_point2d` | struct | 2D point with int16 coordinates; implicit cast to `long_point2d` |
| `world_point3d` | struct | 3D point with int16 coordinates; implicit cast to `long_point3d` |
| `fixed_vector3d` | struct | 3D vector using fixed-point (16.16) components |
| `fixed_point3d` | struct | 3D point using fixed-point (16.16) components |
| `fixed_yaw_pitch` | struct | Orientation representation: yaw and pitch angles in fixed-point |
| `world_location3d` | struct | Complete spatial state: 3D position, polygon index, yaw/pitch angles, 3D velocity vector |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cosine_table` | extern `short*` | global | Precomputed cosine lookup table (defined in WORLD.C) |
| `sine_table` | extern `short*` | global | Precomputed sine lookup table (defined in WORLD.C) |

## Key Functions / Methods

### normalize_angle
- **Signature:** `static inline angle normalize_angle(angle theta)`
- **Purpose:** Normalize an angle to the valid range [0, NUMBER_OF_ANGLES)
- **Inputs:** `theta` (possibly non-normalized angle)
- **Outputs/Return:** Normalized angle in [0, 512)
- **Side effects:** None
- **Calls:** Expands inline `NORMALIZE_ANGLE(theta)` macro using bitwise AND
- **Notes:** Uses bitwise masking rather than modulo for speed (inlined per comment from Loren Petrich, Feb 2000)

### cross_product_k
- **Signature:** `inline Sint64 cross_product_k(long_vector2d a, long_vector2d b)`
- **Purpose:** Compute the z-component of the 3D cross product of two 2D vectors (scalar)
- **Inputs:** Two `long_vector2d` vectors
- **Outputs/Return:** `Sint64` (int64) result: `a.i * b.j - a.j * b.i`
- **Side effects:** None
- **Calls:** None (pure arithmetic)
- **Notes:** Uses int64 intermediate to avoid int32 overflow

### rotate_point2d / rotate_point3d
- **Signature:** `world_point2d *rotate_point2d(world_point2d *point, world_point2d *origin, angle theta)`; 3D version adds `angle phi`
- **Purpose:** Rotate a point about an origin by angles theta (yaw) and/or phi (pitch)
- **Inputs:** Point and origin (as pointers); rotation angles
- **Outputs/Return:** Pointer to mutated `point` (in-place rotation)
- **Side effects:** Modifies input point structure
- **Calls:** Defined in WORLD.C; uses cosine/sine tables
- **Notes:** In-place mutation pattern; 3D version requires two angles

### translate_point2d / translate_point3d
- **Signature:** `world_point2d *translate_point2d(world_point2d *point, world_distance distance, angle theta)`
- **Purpose:** Move a point along a direction vector defined by distance and angle(s)
- **Inputs:** Point, distance magnitude, angle(s) indicating direction
- **Outputs/Return:** Pointer to mutated point
- **Side effects:** Modifies input point in-place
- **Calls:** Defined in WORLD.C; uses trig tables
- **Notes:** 3D variant includes pitch for 3D translation direction

### transform_point2d / transform_point3d
- **Signature:** `world_point2d *transform_point2d(world_point2d *point, world_point2d *origin, angle theta)`
- **Purpose:** Apply a combined rotation about an origin and translation (or rotation-then-translation in local frame)
- **Inputs:** Point, origin, angle(s)
- **Outputs/Return:** Pointer to mutated point
- **Side effects:** Modifies input point in-place
- **Calls:** Likely decomposes to rotation + translation in WORLD.C
- **Notes:** Combines geometric transformations; 3D variant exists

### arctangent
- **Signature:** `angle arctangent(int32 x, int32 y)`
- **Purpose:** Compute the angle (direction) from y/x, returning angle in [0, NUMBER_OF_ANGLES)
- **Inputs:** `x`, `y` (int32; can be large for long-distance)
- **Outputs/Return:** Angle in [0, 512)
- **Side effects:** None
- **Calls:** Defined in WORLD.C; uses internal tangent table
- **Notes:** Made "long-distance friendly" (Feb 2000) to handle large int32 values without overflow

### Random Functions
- **`set_random_seed(uint16 seed)`:** Initialize random number generator with seed
- **`get_random_seed(void) ΓåÆ uint16`:** Retrieve current random seed state
- **`global_random(void) ΓåÆ uint16`:** Generate next global pseudo-random number
- **`local_random(void) ΓåÆ uint16`:** Generate next locally-scoped random number (purpose of scope distinction not inferable from header)

### Distance Functions
- **`guess_distance2d(world_point2d *p0, world_point2d *p1) ΓåÆ world_distance`:** Fast approximate 2D distance using `GUESS_HYPOTENUSE` (adds max(dx,dy) + 0.5*min(dx,dy))
- **`distance2d(world_point2d *p0, world_point2d *p1) ΓåÆ world_distance`:** Exact 2D Euclidean distance (calls `isqrt()`)
- **`distance3d(world_point3d *p0, world_point3d *p1) ΓåÆ world_distance`:** Exact 3D Euclidean distance
- **`isqrt(uint32 x) ΓåÆ int32`:** Integer square root for distance calculations

### Overflow Handling
- **`long_to_overflow_short_2d(long_vector2d& LVec, world_point2d& WVec, uint16& flags)`:** Pack upper 4 bits of long int32 coordinates into flags byte
- **`overflow_short_to_long_2d(world_point2d& WVec, uint16& flags, long_vector2d& LVec)`:** Unpack flags into high bits of long coordinates
- **`transform_overflow_point2d(..., uint16 *flags)`:** Perform transform while preserving overflow state
- **Purpose:** Kludge to support long-distance calculations beyond int16 range by storing overflow bits separately (per Loren Petrich comments)

## Control Flow Notes
This is a pure declaration header with inline implementations and macros. It is foundational for all spatial calculations in the engine. Initialization occurs via `build_trig_tables()` (called once at startup). Spatial transformations (rotation, translation) and distance checks are used during entity updates and collision queries. Random functions service gameplay logic. The dual `world_*` / `long_*` hierarchy allows code to choose precision trade-offs: world-distance for compact storage and local calculations, long for intermediate high-precision arithmetic.

## External Dependencies
- **Includes:** `cstypes.h` (for base integer types and `_fixed`), `<tuple>` (for `std::tie` in `world_location3d` equality operators)
- **Defined elsewhere:** `cosine_table`, `sine_table` (extern symbols from WORLD.C); all function implementations (`build_trig_tables`, `rotate_point*`, `translate_point*`, `transform_point*`, `arctangent`, `global_random`, `local_random`, `distance*`, `isqrt`, overflow functions) are in WORLD.C
