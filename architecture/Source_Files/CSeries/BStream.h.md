# Source_Files/CSeries/BStream.h

## File Purpose
Provides serialization/deserialization classes for reading from and writing to `std::streambuf` objects. Replaces the older AStream interface with support for endianness-aware binary I/O (Big-Endian). Used for persistent game data and resource loading.

## Core Responsibilities
- Wrap `std::streambuf` pointers for abstraction over underlying I/O sources (files, memory, pipes, etc.)
- Provide typed extraction (`operator>>`) and insertion (`operator<<`) for primitive types
- Support Big-Endian serialization via concrete `BIStreamBE` and `BOStreamBE` classes
- Handle 8-bit types (no conversion) and multi-byte types (endianness-aware)
- Query stream position and bounds (`tellg()`, `maxg()`, `tellp()`, `maxp()`)
- Support raw byte operations (`read()`, `write()`, `ignore()`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `basic_bstream` | class | Abstract base for all stream wrappers; manages `streambuf` ownership |
| `BIStream` | class | Abstract input stream; defines extraction operators as pure virtual for multi-byte types |
| `BIStreamBE` | class | Concrete Big-Endian input stream; implements `operator>>` for int16, uint16, int32, uint32, double |
| `BOStream` | class | Abstract output stream; defines insertion operators as pure virtual for multi-byte types |
| `BOStreamBE` | class | Concrete Big-Endian output stream; implements `operator<<` for int16, uint16, int32, uint32, double |

## Global / File-Static State
None.

## Key Functions / Methods

### basic_bstream constructor & rdbuf()
- **Signature:** `explicit basic_bstream(std::streambuf* sb)`; `rdbuf()`, `rdbuf(std::streambuf* new_sb)`
- **Purpose:** Initialize stream wrapper and get/set underlying streambuf
- **Notes:** Uses explicit keyword to prevent implicit conversions

### BIStream::operator>>() (uint8, int8, double, int16, uint16, int32, uint32)
- **Purpose:** Extract values from input stream; 8-bit versions use non-virtual implementation; multi-byte overrides in `BIStreamBE`
- **Inputs:** Reference to target variable
- **Outputs/Return:** Reference to `*this` for chaining
- **Side effects:** Advances stream position; may throw `std::ios_base::failure`
- **Calls:** Delegates multi-byte ops to concrete subclass implementations

### BIStream::tellg(), maxg()
- **Signature:** `std::streampos tellg()`, `maxg()`
- **Purpose:** Query current read position and stream bounds
- **Notes:** Defined in `.cpp`; used for validation and seek operations

### BIStream::read(), ignore()
- **Signature:** `read(char* s, std::streamsize n)`, `ignore(std::streamsize n)`
- **Purpose:** Raw byte operations; read n bytes into buffer or skip n bytes
- **Notes:** Defined in `.cpp`

### BOStream::operator<<() (uint8, int8, double, int16, uint16, int32, uint32)
- **Purpose:** Insert values into output stream; mirrors BIStream pattern
- **Outputs/Return:** Reference to `*this` for chaining
- **Side effects:** Advances write position; may throw

### BOStream::write()
- **Signature:** `write(const char* s, std::streamsize n)`
- **Purpose:** Write n raw bytes from buffer
- **Notes:** Defined in `.cpp`

## Control Flow Notes
This is a utility library, not part of the game loop. Called during:
- **Initialization/shutdown:** Loading config, game state, resources
- **Persistence:** Saving/loading game saves, maps, assets
- **Network I/O:** Potential serialization for multiplayer (not visible in this header)

The virtual `operator>>` / `operator<<` pattern defers endianness conversion to concrete subclasses, allowing future implementations (e.g., Little-Endian variants).

## External Dependencies
- `cseries.h` (which includes `cstypes.h` ΓÇö defines `uint8`, `int8`, `int16`, `uint16`, `int32`, `uint32`)
- `<streambuf>` (STL; abstract I/O buffer)
- `std::ios_base::failure` (exception type)
