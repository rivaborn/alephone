# Source_Files/Files/AStream.h

## File Purpose
Provides typed, endian-aware binary serialization/deserialization streams for game data. Replaces the less-clear `Packing.h` from AlephOne with explicit type safety and compile-time endian selection via subclassing rather than preprocessor directives.

## Core Responsibilities
- Define templated base stream class with state/bounds tracking and exception handling
- Provide input stream classes (`AIStream*`) for binary deserialization with operator>>
- Provide output stream classes (`AOStream*`) for binary serialization with operator<<
- Support both Big Endian (BE) and Little Endian (LE) byte order conversion
- Track stream position, read bounds, error states (good/bad/fail), and exception masks
- Enable safe array/buffer reading and writing via template `read()` / `write()` methods

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `AStream::_Aiostate` | enum | Stream state bit flags (goodbit, badbit, failbit) |
| `AStream::iostate` | typedef | Alias for stream state flags |
| `AStream::failure` | class | Exception thrown on stream errors |
| `AStream::basic_astream<T>` | template class | Core stream base with state, bounds, position tracking |
| `AIStream` | class | Input stream base; abstract for 16/32-bit operators |
| `AIStreamBE` / `AIStreamLE` | classes | Concrete Big/Little Endian input streams |
| `AOStream` | class | Output stream base; abstract for 16/32-bit operators |
| `AOStreamBE` / `AOStreamLE` | classes | Concrete Big/Little Endian output streams |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AStream::badbit` | iostate | namespace static | Flag: stream corruption or unrecoverable error |
| `AStream::failbit` | iostate | namespace static | Flag: logical operation failed |
| `AStream::goodbit` | iostate | iostate | Flag: stream in good state (value 0) |

## Key Functions / Methods

### basic_astream::basic_astream (constructor)
- **Signature:** `basic_astream(T* __stream, uint32 __length, uint32 __offset)`
- **Purpose:** Initialize stream with buffer bounds and starting position.
- **Inputs:** pointer to buffer start, buffer length, initial offset into buffer.
- **Outputs/Return:** None (constructor).
- **Side effects:** Sets `_M_stream_begin`, `_M_stream_end`, `_M_stream_pos`, `_M_state` (goodbit by default), `_M_exception` (failbit by default).
- **Notes:** Sets badbit if `__offset > __length` (invalid initial position).

### basic_astream::bound_check
- **Signature:** `bool bound_check(uint32 __delta)`
- **Purpose:** Verify that advancing stream position by `__delta` bytes stays within bounds.
- **Notes:** Implementation in .cpp; called before each read/write to prevent buffer overrun.

### AIStream::operator>> (extraction, multiple overloads)
- **Signature:** `AIStream& operator>>(uint8|int8|bool|uint16|int16|uint32|int32 &__value)`
- **Purpose:** Deserialize typed value from input stream, advancing position.
- **Inputs:** Reference to output variable.
- **Outputs/Return:** `*this` (for chaining).
- **Side effects:** Increments `_M_stream_pos`, may set failbit if bounds exceeded.
- **Notes:** 8-bit overloads (uint8, int8) defined in base class. 16/32-bit overloads are pure virtual in base, implemented by endian subclasses (BE/LE perform byte-swapping).

### AIStream::read (template)
- **Signature:** `AIStream& read(T* __list, uint32 __count)` (template)
- **Purpose:** Deserialize array of typed elements by invoking operator>> in a loop.
- **Inputs:** Pointer to array, element count.
- **Outputs/Return:** `*this`.
- **Side effects:** Advances position by count ├ù sizeof(T).
- **Notes:** Trivial wrapper enabling `stream.read(array, N)` syntax.

### AOStream::operator<< (insertion, multiple overloads)
- **Signature:** `AOStream& operator<<(uint8|int8|bool|uint16|int16|uint32|int32 __value)`
- **Purpose:** Serialize typed value to output stream, advancing position.
- **Inputs:** Value to write.
- **Outputs/Return:** `*this`.
- **Side effects:** Increments `_M_stream_pos`, may set failbit if bounds exceeded.
- **Notes:** 8-bit overloads in base; 16/32-bit pure virtual, implemented by endian subclasses.

### AOStream::write (template)
- **Signature:** `AOStream& write(T* __list, uint32 __count)`
- **Purpose:** Serialize array of typed elements by invoking operator<< in a loop.
- **Outputs/Return:** `*this`.
- **Notes:** Mirrors AIStream::read pattern.

## Control Flow Notes
Not part of frame/render loop directly. Used by game systems (networking, save/load, replay) to pack/unpack game state and data structures into byte buffers. Endian subclass (BE or LE) chosen at instantiation based on required format.

## External Dependencies
- `<string>`, `<exception>` ΓÇô Standard C++ for `failure` exception class
- `"cstypes.h"` ΓÇô Custom fixed-width integer typedefs (`uint8`, `int16`, `uint32`, etc.)
