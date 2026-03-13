# Source_Files/CSeries/BStream.cpp

## File Purpose
Binary stream serialization/deserialization layer providing type-safe input/output for primitive data with endianness support. Designed to replace AStream and wrap standard C++ streambuf for game data I/O.

## Core Responsibilities
- Deserialize primitive types (8ΓÇô64-bit integers, doubles) from input streams
- Serialize primitive types to output streams
- Handle big-endian byte swapping via SDL2
- Track stream read/write positions and bounds
- Enforce serialization bounds with exception-based error handling
- Support method chaining for fluent stream operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| BIStream | class | Abstract binary input stream; base for endianness variants |
| BIStreamBE | class | Concrete big-endian input stream implementation |
| BOStream | class | Abstract binary output stream; base for endianness variants |
| BOStreamBE | class | Concrete big-endian output stream implementation |

## Global / File-Static State
None.

## Key Functions / Methods

### BIStream::tellg()
- **Signature:** `std::streampos tellg() const`
- **Purpose:** Query current read position
- **Inputs:** None
- **Outputs/Return:** Current position in the input stream
- **Calls:** `rdbuf()->pubseekoff(0, cur, in)`

### BIStream::maxg()
- **Signature:** `std::streampos maxg() const`
- **Purpose:** Determine stream end position without changing current position
- **Inputs:** None
- **Outputs/Return:** End position; restores original position
- **Side effects:** Temporarily seeks to end, then seeks back
- **Calls:** `rdbuf()->pubseekoff()` twice

### BIStream::read()
- **Signature:** `BIStream& read(char *s, std::streamsize n)`
- **Purpose:** Read n raw bytes into buffer; throws on underread
- **Inputs:** Buffer pointer, byte count
- **Outputs/Return:** Reference to *this
- **Side effects:** Throws `failure` if fewer than n bytes available
- **Calls:** `rdbuf()->sgetn()`

### BIStream::operator>>(uint8&) / operator>>(int8&)
- **Purpose:** Deserialize single bytes (no endian conversion needed)
- **Inputs:** Reference to value
- **Outputs/Return:** Reference to *this
- **Calls:** `read()`

### BIStreamBE::operator>>(uint16&), operator>>(int16&), operator>>(uint32&), operator>>(int32&)
- **Purpose:** Deserialize multi-byte integers with big-endian conversion
- **Inputs:** Reference to value
- **Outputs/Return:** Reference to *this
- **Side effects:** Swaps byte order via `SDL_SwapBE16/32`
- **Calls:** `read()`, SDL endian functions

### BIStreamBE::operator>>(double&)
- **Purpose:** Deserialize 8-byte double with big-endian conversion
- **Inputs:** Reference to double
- **Outputs/Return:** Reference to *this
- **Side effects:** Reads as Uint64, swaps, then memcpys to double to avoid strict-aliasing issues
- **Calls:** `read()`, `SDL_SwapBE64()`, `memcpy()`

### BOStream::write() / BOStream::tellp() / BOStream::maxp()
- **Purpose:** Mirror of BIStream for output (write bytes, query position, get end)
- **Calls:** `rdbuf()->sputn()`, `pubseekoff()`

### BOStreamBE::operator<<() (all variants)
- **Purpose:** Serialize integers/doubles with big-endian byte swapping
- **Inputs:** Value by copy
- **Outputs/Return:** Reference to *this
- **Side effects:** Swaps bytes before writing; double uses Uint64 intermediate
- **Calls:** `write()`, SDL endian functions

## Control Flow Notes
Not part of frame/render loop. Used on-demand for:
- Loading/saving game resources and state (likely from `MapLoader`, `SaveGame`, etc.)
- Serializing network packets or asset bundles
- Decoupling file I/O from game logic via streambuf abstraction

## External Dependencies
- **SDL2/SDL_endian.h** ΓÇô `SDL_SwapBE16()`, `SDL_SwapBE32()`, `SDL_SwapBE64()` for platform-independent endian conversion
- **std::streambuf** ΓÇô underlying buffer management (standard C++)
- **cseries.h** ΓÇô project type definitions (uint8, int8, Uint64, etc.)
