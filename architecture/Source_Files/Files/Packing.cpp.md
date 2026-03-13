# Source_Files/Files/Packing.cpp

## File Purpose

Implements binary serialization primitives for converting 16-bit and 32-bit integers to/from byte streams. Supports both big-endian (BE) and little-endian (LE) byte orders with sign-aware handling for signed and unsigned variants. Used by Marathon's game engine to pack/unpack network and file data.

## Core Responsibilities

- Convert big-endian byte streams to 16-bit/32-bit unsigned integers
- Convert big-endian byte streams to 16-bit/32-bit signed integers
- Convert 16-bit/32-bit integers to big-endian byte streams
- Convert little-endian byte streams to 16-bit/32-bit unsigned integers
- Convert little-endian byte streams to 16-bit/32-bit signed integers
- Convert 16-bit/32-bit integers to little-endian byte streams
- Advance stream pointers by consumed/written byte count

## Key Types / Data Structures

None.

## Global / File-Static State

None.

## Key Functions / Methods

### StreamToValueBE (uint16 variant)
- **Signature:** `void StreamToValueBE(uint8* &Stream, uint16 &Value)`
- **Purpose:** Unpack two bytes from stream as big-endian unsigned 16-bit integer.
- **Inputs:** Stream pointer (reference), destination reference
- **Outputs/Return:** Value populated with unpacked uint16; Stream advanced by 2 bytes
- **Side effects:** Stream pointer modified (advanced by 2)
- **Calls:** None (pointer dereference and post-increment only)
- **Notes:** Both byte loads are explicitly cast to uint16 before shifting to avoid sign-extension issues. First byte is high byte (shifted left 8).

### StreamToValueBE (int16 variant)
- **Signature:** `void StreamToValueBE(uint8* &Stream, int16 &Value)`
- **Purpose:** Unpack two bytes from stream as big-endian signed 16-bit integer.
- **Inputs:** Stream pointer (reference)
- **Outputs/Return:** Value populated with unpacked int16 (via reinterpret cast from uint16)
- **Side effects:** Stream pointer advanced by 2 bytes
- **Calls:** `StreamToValueBE(Stream, UValue)` [uint16 overload]

### StreamToValueBE (uint32 variant)
- **Signature:** `void StreamToValueBE(uint8* &Stream, uint32 &Value)`
- **Purpose:** Unpack four bytes from stream as big-endian unsigned 32-bit integer.
- **Inputs:** Stream pointer (reference)
- **Outputs/Return:** Value populated with unpacked uint32; Stream advanced by 4 bytes
- **Side effects:** Stream pointer modified (advanced by 4)

### StreamToValueBE (int32 variant)
- **Signature:** `void StreamToValueBE(uint8* &Stream, int32 &Value)`
- **Purpose:** Unpack four bytes as big-endian signed 32-bit integer via unsigned intermediate.
- **Outputs/Return:** Value populated; Stream advanced by 4 bytes
- **Calls:** `StreamToValueBE(Stream, UValue)` [uint32 overload]

### ValueToStreamBE (uint16 variant)
- **Signature:** `void ValueToStreamBE(uint8* &Stream, uint16 Value)`
- **Purpose:** Pack unsigned 16-bit integer into stream as big-endian.
- **Inputs:** Stream pointer (reference), value to pack
- **Side effects:** Stream advanced by 2 bytes; bytes written to stream
- **Notes:** High byte written first; second write masks to lower 8 bits.

### ValueToStreamBE (int16 variant)
- **Signature:** `void ValueToStreamBE(uint8* &Stream, int16 Value)`
- **Purpose:** Pack signed 16-bit integer into stream as big-endian via uint16 cast.
- **Calls:** `ValueToStreamBE(Stream, uint16(Value))`

### ValueToStreamBE (uint32 variant)
- **Signature:** `void ValueToStreamBE(uint8* &Stream, uint32 Value)`
- **Purpose:** Pack unsigned 32-bit integer into stream as big-endian (4 bytes).
- **Side effects:** Stream advanced by 4 bytes

### ValueToStreamBE (int32 variant)
- **Signature:** `void ValueToStreamBE(uint8* &Stream, int32 Value)`
- **Purpose:** Pack signed 32-bit integer as big-endian via uint32 cast.
- **Calls:** `ValueToStreamBE(Stream, uint32(Value))`

### StreamToValueLE (uint16, int16, uint32, int32 variants)
- **Purpose:** Identical to StreamToValueBE variants but reverse byte order (little-endian). Byte0 is low byte, Byte1 is high byte for 16-bit; Byte3 becomes high byte for 32-bit.
- **Side effects:** Stream advanced identically (2 or 4 bytes)

### ValueToStreamLE (uint16, int16, uint32, int32 variants)
- **Purpose:** Identical to ValueToStreamBE variants but write bytes in reverse order (low byte first for LE vs. high byte first for BE).
- **Side effects:** Stream advanced identically (2 or 4 bytes)

## Control Flow Notes

These are primitive utility functions called during game data deserialization (unpacking) and serialization (packing). The header (Packing.h) defines preprocessor macros (`StreamToValue`, `ValueToStream`) that resolve to either the BE or LE variants based on `PACKED_DATA_IS_BIG_ENDIAN` or `PACKED_DATA_IS_LITTLE_ENDIAN` defines. Higher-level template functions in the header (e.g., `StreamToList`, `ListToStream`) iterate over collections using these primitives.

## External Dependencies

- **cseries.h** ΓåÆ includes cstypes.h (defines uint8, uint16, int16, uint32, int32)
- **Packing.h** ΓåÆ header declaring extern function prototypes and macro dispatch logic
- **DEFINE_PACKING_INTERNAL** ΓåÆ signals to Packing.h not to inline certain utilities (implementation moved to .cpp to resolve inlining issues per Aug 2002 note)
