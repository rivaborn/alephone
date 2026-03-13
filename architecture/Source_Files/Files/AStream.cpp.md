# Source_Files/Files/AStream.cpp

## File Purpose
Implements serialization/deserialization stream classes for network communication and data persistence. Provides explicit big-endian and little-endian byte ordering support with bounds checking and exception handling.

## Core Responsibilities
- Implement extraction operators (`>>`) for input stream deserialization across signed/unsigned types
- Implement insertion operators (`<<`) for output stream serialization
- Provide separate big-endian and little-endian implementations for multi-byte types
- Perform bounds checking on stream positions with optional exception throwing
- Define exception class for serialization failures with proper resource cleanup

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| AIStream | class | Base input stream for deserialization (derived from `basic_astream<const uint8>`) |
| AIStreamBE | class | Big-endian input stream, overrides 16/32-bit extraction operators |
| AIStreamLE | class | Little-endian input stream, overrides 16/32-bit extraction operators |
| AOStream | class | Base output stream for serialization (derived from `basic_astream<uint8>`) |
| AOStreamBE | class | Big-endian output stream, overrides 16/32-bit insertion operators |
| AOStreamLE | class | Little-endian output stream, overrides 16/32-bit insertion operators |
| AStream::failure | class | Exception class for serialization bound violations |

## Global / File-Static State
None.

## Key Functions / Methods

### AIStream::operator>>(uint8 &value)
- Signature: `AIStream& operator>>(uint8 &value)`
- Purpose: Extract single unsigned byte from input stream
- Inputs: Reference to `uint8` to populate
- Outputs/Return: Reference to `*this` (supports chaining)
- Side effects: Advances `_M_stream_pos` by 1 byte if bounds check passes; may set failbit state
- Calls: `bound_check(1)`
- Notes: Base implementation for all other operators; all other types delegate to unsigned versions

### AIStream::operator>>(int8 &value)
- Signature: `AIStream& operator>>(int8 &value)`
- Purpose: Extract signed byte (delegates to uint8 then casts)
- Outputs/Return: Reference to `*this`
- Calls: `operator>>(uint8 &)`
- Notes: Sign conversion happens after deserialization

### AIStreamBE::operator>>(uint16 &value)
- Signature: `AIStream& operator>>(uint16 &value)` (overrides virtual)
- Purpose: Extract 2 bytes in big-endian order (MSB first)
- Side effects: Advances stream position by 2 bytes; constructs value as `(Byte0 << 8) | Byte1`
- Calls: `bound_check(2)`
- Notes: Zero-extends individual bytes before combining; both bytes read as unsigned

### AIStreamBE::operator>>(uint32 &value)
- Signature: `AIStream& operator>>(uint32 &value)`
- Purpose: Extract 4 bytes in big-endian order
- Side effects: Advances stream position by 4 bytes; constructs value as `(B0 << 24) | (B1 << 16) | (B2 << 8) | B3`
- Calls: `bound_check(4)`
- Notes: Pattern matches uint16 but with 4 bytes

### AIStreamLE equivalents
- Identical signatures to `AIStreamBE`, but byte order is reversed: `(Byte1 << 8) | Byte0` for uint16; `(B3 << 24) | (B2 << 16) | (B1 << 8) | B0` for uint32

### AIStream::read(char *ptr, uint32 count)
- Signature: `AIStream& read(char *ptr, uint32 count)`
- Purpose: Bulk read `count` bytes from stream into buffer
- Inputs: Pointer to buffer, byte count
- Outputs/Return: Reference to `*this`
- Side effects: Copies bytes via `memcpy` and advances stream position; calls `bound_check(count)`
- Notes: Provides alternative to repeated `operator>>` calls for untyped data

### AIStream::ignore(uint32 count)
- Signature: `AIStream& ignore(uint32 count)`
- Purpose: Skip `count` bytes without reading
- Side effects: Advances stream position only; checks bounds
- Notes: No-op if bounds check fails

### AOStream::operator<<(uint8 value)
- Signature: `AOStream& operator<<(uint8 value)`
- Purpose: Insert single unsigned byte into output stream
- Inputs: `uint8` value
- Outputs/Return: Reference to `*this`
- Side effects: Writes byte at `_M_stream_pos++` if bounds check passes
- Calls: `bound_check(1)`

### AOStreamBE::operator<<(uint16 value)
- Signature: `AOStream& operator<<(uint16 value)`
- Purpose: Insert 2 bytes in big-endian order (MSB first)
- Side effects: Writes two bytes as `uint8(value >> 8)` then `uint8(value)`, advances position by 2
- Calls: `bound_check(2)`

### AOStreamBE::operator<<(uint32 value)
- Signature: `AOStream& operator<<(uint32 value)`
- Purpose: Insert 4 bytes in big-endian order
- Side effects: Writes bytes in order: `(value >> 24)`, `(value >> 16)`, `(value >> 8)`, `value`
- Calls: `bound_check(4)`

### AOStreamLE equivalents
- Identical signatures to `AOStreamBE`, but byte order reversed: LSB first

### AStream::basic_astream<T>::bound_check(uint32 delta)
- Signature: `bool bound_check(uint32 delta)` (template method)
- Purpose: Verify `delta` bytes are available from current position to stream end
- Inputs: Number of bytes to validate
- Outputs/Return: `true` if bounds satisfied; `false` if violated
- Side effects: Sets failbit state on failure; throws `AStream::failure` if failbit exception is enabled
- Notes: Called by all stream operations; exception throwing controlled by `exceptions()` mask

### AStream::failure::failure(const std::string& str)
- Signature: `failure(const std::string& str) noexcept`
- Purpose: Constructor; allocates and stores error message
- Side effects: Calls `strdup()` to duplicate string; `_M_name` owns allocated memory
- Notes: Noexcept despite allocation (caller must handle alloc failure elsewhere)

### AStream::failure::~failure()
- Signature: `~failure() noexcept`
- Purpose: Clean up allocated error message
- Side effects: Calls `free()` on `_M_name` if non-null; nullifies pointer
- Notes: Safe to call multiple times

### AStream::failure::what() const
- Signature: `const char* what() const noexcept` (overrides `std::exception`)
- Purpose: Return C-string error message
- Outputs/Return: Pointer to internal `_M_name` buffer
- Notes: Returns null if message was not set (copy constructor may not populate `_M_name`)

## Control Flow Notes
Not inferable from this implementation file. Header declares these as abstract serialization interfaces used by higher-level game engine code to pack/unpack network packets or save-game data.

## External Dependencies
- `<string.h>`: `memcpy()`, `strdup()`, `free()`
- `<string>`: `std::string` (used in exception constructor)
- `<exception>`: `std::exception` (base class for `failure`)
- `"AStream.h"`: Class declarations and `basic_astream<T>` template
- `"cstypes.h"`: Type aliases (`uint8`, `int8`, `uint16`, `int16`, `uint32`, `int32`)
- Conditional compilation guard: `#if !defined(DISABLE_NETWORKING)`
