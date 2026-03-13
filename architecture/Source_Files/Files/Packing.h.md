# Source_Files/Files/Packing.h

## File Purpose
Provides serialization utilities to convert between in-memory data structures (with native alignment) and packed big-endian byte streams, as used by the Marathon series. Handles endianness conversion and stream pointer advancement during pack/unpack operations.

## Core Responsibilities
- Unpack numerical values (16/32-bit int/uint) from byte streams via `StreamToValue()`
- Pack numerical values into byte streams via `ValueToStream()`
- Serialize/deserialize arrays of values via `StreamToList()` and `ListToStream()`
- Handle raw byte block packing/unpacking without interpretation
- Support configurable endianness (big-endian default, overridable to little-endian)
- Maintain stream pointer advancement across all operations

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### StreamToValue
- Signature: `void StreamToValue(uint8* &Stream, T& Value)` where T Γêê {uint16, int16, uint32, int32}
- Purpose: Unpack a single numerical value from a byte stream
- Inputs: Stream pointer (passed by reference), value reference
- Outputs/Return: Modifies value parameter, advances stream pointer
- Side effects: Performs endianness conversion (if configured); advances stream pointer
- Calls: Not inferable (extern, defined in Packing.cpp)
- Notes: Four overloads for different integer types; dispatched via macro to BE/LE variants

### ValueToStream
- Signature: `void ValueToStream(uint8* &Stream, T Value)` where T Γêê {uint16, int16, uint32, int32}
- Purpose: Pack a single numerical value into a byte stream
- Inputs: Stream pointer (passed by reference), value (by value)
- Outputs/Return: Advances stream pointer
- Side effects: Performs endianness conversion (if configured); writes to stream
- Calls: Not inferable (extern, defined in Packing.cpp)
- Notes: Four overloads for different integer types

### StreamToList
- Signature: `template<class T> inline static void StreamToList(uint8* &Stream, T* List, size_t Count)`
- Purpose: Unpack an array of numerical values from a stream
- Inputs: Stream pointer, array pointer, element count
- Outputs/Return: Modifies array in-place, advances stream pointer
- Side effects: Calls `StreamToValue` repeatedly
- Calls: `StreamToValue`

### ListToStream
- Signature: `template<class T> inline static void ListToStream(uint8* &Stream, const T* List, size_t Count)`
- Purpose: Pack an array of numerical values into a stream
- Inputs: Stream pointer, array pointer (const), element count
- Outputs/Return: Advances stream pointer
- Side effects: Calls `ValueToStream` repeatedly
- Calls: `ValueToStream`

### StreamToBytes
- Signature: `inline static void StreamToBytes(uint8* &Stream, void* Bytes, size_t Count)`
- Purpose: Copy raw bytes from stream to buffer without interpretation
- Inputs: Stream pointer, destination buffer, byte count
- Outputs/Return: Advances stream pointer
- Calls: `memcpy`

### BytesToStream
- Signature: `inline static void BytesToStream(uint8* &Stream, const void* Bytes, size_t Count)`
- Purpose: Copy raw bytes from buffer into stream
- Inputs: Stream pointer, source buffer (const), byte count
- Outputs/Return: Advances stream pointer
- Calls: `memcpy`

## Control Flow Notes
Utility header with no flow control. Likely used during game asset loading (maps, sprites, sounds) to deserialize packed binary formats into runtime structures, and during save operations for serialization.

## External Dependencies
- `cstypes.h`: Provides sized integer typedefs (uint8, int16, uint16, int32, uint32)
- `memcpy`: Standard C library for raw byte copying
- `Packing.cpp`: Contains implementations of `StreamToValue` and `ValueToStream` overloads

## Notes
- **Endianness configuration**: Defaults to big-endian; preprocessor allows override via `PACKED_DATA_IS_LITTLE_ENDIAN` (but not both simultaneously)
- **Stream pointer semantics**: All functions take stream pointer by reference and advance it; caller must preserve base pointer for multi-call sequences
- **Template instantiation**: `StreamToList`/`ListToStream` are inline templates compiled in this header unless `PACKING_INTERNAL` is defined
