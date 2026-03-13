# Source_Files/CSeries/csmacros.h

## File Purpose
Utility header providing macros and inline functions for the Aleph One game engine. Includes comparison operations, flag bit manipulation, bounds checking, and type-safe memory wrappers.

## Core Responsibilities
- Provide MIN/MAX comparison and value clamping (PIN) with compatibility variants
- Implement flag testing and manipulation for 16-bit and 32-bit fields
- Calculate rectangle dimensions from coordinate structs
- Offer bounds-checked array access via templates (C++ only)
- Provide type-safe memcpy/memset wrappers for object operations (C++ only)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### NextPowerOfTwo
- Signature: `static inline int NextPowerOfTwo(int n)`
- Purpose: Return smallest power of two ΓëÑ n
- Inputs: `n` ΓÇô integer value
- Outputs/Return: power of two
- Side effects: none
- Calls: (bitwise only)
- Notes: Iterative bit-shift; assumes 32-bit int

### GetMemberWithBounds
- Signature: `template<class T> T* GetMemberWithBounds(T* Array, const size_t Index, const size_t Number)`
- Purpose: Safe bounds-checked array element access
- Inputs: array pointer, index, array size
- Outputs/Return: pointer to element if in bounds, NULL otherwise
- Side effects: none
- Calls: (none)
- Notes: Caller must check NULL; C++ only

### Template Memory Operations
- **obj_copy / objlist_copy**: Type-safe memcpy wrappers; eliminate manual sizeof
- **obj_set / objlist_set**: Type-safe memset wrappers; fill objects with byte value
- **obj_clear / objlist_clear**: Zero-initialize single or multiple objects
- **Side effect:** `objlist_*` variants skip operation if count is 0
- **Calls:** `memcpy()`, `memset()` (C++ only)

## Macro Summary
- **Comparison:** `MAX`, `MIN`, `FLOOR`, `CEILING` ΓÇô numeric bounds
- **Clamping:** `M2_PIN`, `A1_PIN`, `PIN` ΓÇô clamp between floor/ceiling; `PIN` switches behavior via `film_profile.inexplicable_pin_change`
- **Utility:** `SGN` (sign function), `RECTANGLE_WIDTH`, `RECTANGLE_HEIGHT`
- **Flags 32-bit:** `FLAG`, `TEST_FLAG32`, `SET_FLAG32`
- **Flags 16-bit:** `FLAG16`, `TEST_FLAG16`, `SET_FLAG16`
- **Generic flags:** `TEST_FLAG`, `SET_FLAG`

## Control Flow Notes
Pure utility header with no game-loop integration. Used throughout engine for low-level operations. The `PIN` macro conditionally branches based on `film_profile.inexplicable_pin_change` to support historical gameplay divergence between Marathon 2 and Marathon Infinity during film playback.

## External Dependencies
- `<string.h>` ΓÇô `memcpy()`, `memset()`
- `FilmProfile.h` ΓÇô `film_profile` global for version-specific behavior; header marked "TERRIBLE" in source comment
