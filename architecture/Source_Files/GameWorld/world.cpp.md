# Source_Files/GameWorld/world.cpp

## File Purpose
Core world geometry and mathematics module for the Aleph One game engine (Marathon mod). Provides trigonometric lookup tables, point transformations (translation/rotation in 2D/3D), distance calculations, angle normalization, and deterministic random number generation. Supports both legacy (Marathon 2) and modern (Aleph One) physics implementations for accurate long-distance calculations and film playback.

## Core Responsibilities
- Build and manage global sine/cosine/tangent lookup tables for fixed-point trigonometry
- Normalize angles to valid range [0, NUMBER_OF_ANGLES) = [0, 512) = [0, 2╧Ç)
- Translate points in 2D/3D space by distance and angle(s)
- Rotate and transform points around origins in 2D/3D (relative/absolute coordinate systems)
- Calculate Euclidean distances between points with overflow clamping
- Compute arctangent angles from (x, y) coordinates via binary search (Aleph One) or linear search (Marathon 2)
- Generate deterministic pseudo-random numbers with separate global and local LFSR seeds
- Support long-distance physics via overflow-bit packing in flags uint16
- Implement integer square root (isqrt) for distance calculations without floating-point

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| cosine_table | int16* (extern) | 512-entry precomputed cosine table, normalized to TRIG_MAGNITUDE |
| sine_table | int16* (extern) | 512-entry precomputed sine table |
| tangent_table | int32* (static) | 512 tangent values for arctangent binary/linear search; INT32_MIN at poles |
| angle | typedef (int16) | [0, 512) representing [0, 2╧Ç) |
| world_distance | typedef (int16) | Signed distance with embedded fractional bits |
| long_vector2d / long_point2d | struct | int32 components for overflow safety |
| long_vector3d / long_point3d | struct | int32 components (i,j,k) or (x,y,z) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| cosine_table, sine_table | int16* | extern | Allocated by build_trig_tables(); read by all angle-based ops |
| tangent_table | int32* | static | Lookup for arctangent search; never exposed publicly |
| random_seed | uint16 | static | Global LFSR state; advanced by global_random() |
| local_random_seed | uint16 | static | Local LFSR state; advanced by local_random() (always starts 0x1) |

## Key Functions / Methods

### translate_point2d / translate_point3d
- **Purpose:** Translate a point by distance and angle(s).
- **Signature:** `world_point2d *translate_point2d(world_point2d *p, world_distance d, angle ╬╕)` (2D) / `world_point3d *translate_point3d(...)` (3D with phi for elevation)
- **Inputs:** Mutable point, distance, theta (normalized), [phi].
- **Outputs/Return:** Modified point pointer.
- **Calls:** normalize_angle, cosine_table[], sine_table[].
- **Notes:** 3D variant computes transformed_distance = distance┬╖cos(phi) for horizontal scale, applies theta rotation in x-y plane, and z += distance┬╖sin(phi).

### rotate_point2d / transform_point2d / transform_point3d
- **Purpose:** Rotate or transform points around origins.
- **Key difference:** rotate returns absolute coords; transform returns relative coords.
- **Calls:** normalize_angle, trig tables.
- **Notes:** Use long_vector2d intermediates for precision. transform_point3d applies theta (x-y plane), then phi (x-z plane); skips phi rotation if phi==0.

### arctangent (dispatcher)
- **Purpose:** Compute atan2(y, x) ΓåÆ angle in [0, 512).
- **Signature:** `angle arctangent(int32 x, int32 y)`
- **Dispatch:** Calls m2_arctangent (linear search) or a1_arctangent (binary search) based on film_profile.long_distance_physics.
- **Notes:** Handles edge cases: (0,0)ΓåÆQUARTER_CIRCLE, pure axes, quadrant mapping.

### m2_arctangent / a1_arctangent
- **Purpose:** Legacy (Marathon 2) vs. modern (Aleph One) arctangent implementations.
- **m2_arctangent:** Linear search with sign-change detection.
- **a1_arctangent:** Octant reduction + binary search; faster and handles long distances.
- **Calls:** tangent_table[].
- **Notes:** a1 normalizes to octant 1 before binary search; m2 searches full QUARTER_CIRCLE.

### distance2d / distance3d / guess_distance2d
- **Purpose:** Compute Euclidean distance or fast approximation.
- **distance2d:** Dispatches between m2_distance2d (clamped to INT16) and a1_distance2d (preserves overflow).
- **distance3d:** Always uses isqrt; clamps to INT16_MAX.
- **guess_distance2d:** Macro-based (max + half-min); ~5% underestimate; no sqrt.
- **Calls:** isqrt().

### isqrt
- **Purpose:** Integer square root via binary algorithm (constant 16 iterations).
- **Signature:** `int32 isqrt(uint32 x)` ΓåÆ floor(sqrt(x)) rounded.
- **Calls:** None.
- **Notes:** Extensively documented algorithm: iterates from bit 15 down to 0, subtracting (r+m) when xΓëÑr+m. Rounds up if final remainder > r.

### global_random / local_random
- **Purpose:** LFSR-based deterministic pseudo-random number generation.
- **Signature:** `uint16 global_random(void)` / `uint16 local_random(void)`
- **Algorithm:** Right-shift by 1; if LSB was set, XOR result with 0xb400 (polynomial tap).
- **Side effects:** Update random_seed / local_random_seed in-place.
- **Notes:** Separate streams; global used for gameplay, local for effects/effects.

### long_to_overflow_short_2d / overflow_short_to_long_2d
- **Purpose:** Pack/unpack 32-bit coords into 16-bit world_point2d + upper 4 bits per axis in flags uint16.
- **Calls:** None.
- **Notes:** Stores bits [16..19] of x/y in flags [15..12]/[11..8]; sign-extends on unpack.

### transform_overflow_point2d
- **Purpose:** Rotate/transform point in compressed long-distance space; result repacked into short.
- **Calls:** normalize_angle, trig tables, long_to_overflow_short_2d.
- **Notes:** LP addition for long-distance view support.

## Control Flow Notes
**Initialization:** `build_trig_tables()` called once at startup. Random seeds set via `set_random_seed()`.  
**Per-frame:** Transform/distance functions called during object placement, collision detection, rendering, and aim calculations. Random functions advance seeds for replay determinism.  
**Film mode:** `film_profile.long_distance_physics` flag selects physics model at runtime, allowing legacy films to replay correctly on newer engine.

## External Dependencies
- **Includes:** cseries.h (platform), world.h (constants, declarations), FilmProfile.h (film_profile global), stdlib.h (malloc), math.h (cos, sin, atan), limits.h (INT16_MAX, INT32_MIN).
- **Macros:** TRIG_SHIFT, TRIG_MAGNITUDE, NUMBER_OF_ANGLES, NORMALIZE_ANGLE, GUESS_HYPOTENUSE, QUARTER_CIRCLE, HALF_CIRCLE, THREE_QUARTER_CIRCLE, EIGHTH_CIRCLE.
- **External globals:** `film_profile` (struct FilmProfile).
