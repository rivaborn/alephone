# Source_Files/CSeries/byte_swapping.cpp

## File Purpose
Implements byte-swapping utilities for converting between endianness representations. Enables the Aleph One engine to handle multi-byte data in consistent byte order on little-endian platforms.

## Core Responsibilities
- Reverse byte order for 2-byte (16-bit) field sequences
- Reverse byte order for 4-byte (32-bit) field sequences
- Support in-place memory transformation for bulk data conversion
- Conditionally compiled to be a no-op on big-endian systems

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### byte_swap_memory
- Signature: `void byte_swap_memory(void *memory, _bs_field type, int fieldcount)`
- Purpose: Reverses the byte order of packed fields in memory in-place, converting between little-endian and big-endian representations.
- Inputs:
  - `memory`: pointer to memory region containing fields to swap
  - `type`: enum value (`_2byte` or `_4byte`) specifying field size
  - `fieldcount`: number of fields to process; negative values skip execution
- Outputs/Return: None (void); mutates `memory` in-place
- Side effects: Directly modifies memory region; no allocation or I/O
- Calls: None
- Notes:
  - Uses byte-level temp-swap algorithm (no intrinsics or built-in byte-swap instructions)
  - Only defined under `ALEPHONE_LITTLE_ENDIAN` preprocessor guard; macro becomes no-op elsewhere
  - No bounds checking; assumes valid contiguous memory
  - For `_2byte`: exchanges bytes at offsets [0] and [1]
  - For `_4byte`: exchanges [0]Γåö[3] and [1]Γåö[2] to fully reverse order
  - Pointer increments (`walk += 2/4`) move by field size in bytes

## Control Flow Notes
Low-level utility called during data I/O operations. Compiled conditionally based on platform endianness (via `cseries.h` macro `ALEPHONE_LITTLE_ENDIAN`).

## External Dependencies
- `cseries.h`: Common game engine types and endianness detection
- `byte_swapping.h`: Function declaration and `_bs_field` enum definition
- `uint8`: Fixed-width byte type (from cstypes.h via cseries.h)
