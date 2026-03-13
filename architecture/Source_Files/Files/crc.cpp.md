# Source_Files/Files/crc.cpp

## File Purpose
Provides CRC (Cyclic Redundancy Check) computation utilities for both files and memory buffers. Implements two algorithms: CRC32 (using polynomial-based lookup table) and CCITT CRC16 (using pre-computed table). Used for integrity verification and checksumming in the game engine.

## Core Responsibilities
- Compute CRC32 checksums for files via FileSpecifier abstraction
- Compute CRC32 checksums for in-memory buffers incrementally
- Compute CCITT CRC16 checksums for in-memory buffers
- Build and manage dynamic lookup table for efficient CRC32 calculation
- Handle chunked file I/O with preserved file position
- Provide both high-level file-based and low-level buffer-based APIs

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `uint32` | typedef | 32-bit unsigned integer for CRC32 values |
| `uint16` | typedef | 16-bit unsigned integer for CCITT CRC16 values |
| `crc_ccitt_table` | static array | Pre-computed 256-entry lookup table for CCITT polynomial 0x1021 |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `crc_table` | `uint32*` | static | Dynamically allocated 256-entry lookup table for CRC32 polynomial 0xEDB88320L; shared across all CRC32 operations in this compilation unit |

## Key Functions / Methods

### calculate_crc_for_file
- **Signature:** `uint32 calculate_crc_for_file(FileSpecifier& File)`
- **Purpose:** Public entry point; compute CRC32 for an entire file specified by path
- **Inputs:** `FileSpecifier` reference (file path abstraction)
- **Outputs/Return:** `uint32` CRC32 value; returns 0 on failure
- **Side effects:** Opens file, reads entire contents, builds/frees CRC table, modifies file position then restores it
- **Calls:** `File.Open()`, `calculate_crc_for_opened_file()`, `OFile.Close()`
- **Notes:** Convenience wrapper; delegates to `calculate_crc_for_opened_file()`

### calculate_crc_for_opened_file
- **Signature:** `uint32 calculate_crc_for_opened_file(OpenedFile& OFile)`
- **Purpose:** Public entry point; compute CRC32 for an already-open file handle
- **Inputs:** `OpenedFile` reference (abstract file handle)
- **Outputs/Return:** `uint32` CRC32 value; returns 0 on failure
- **Side effects:** Allocates `BUFFER_SIZE` (1024-byte) buffer on heap, builds/frees CRC table
- **Calls:** `build_crc_table()`, `new[]`, `calculate_file_crc()`, `delete[]`, `free_crc_table()`
- **Notes:** Manages buffer lifecycle; does not modify file position (delegated to `calculate_file_crc()`)

### calculate_data_crc
- **Signature:** `uint32 calculate_data_crc(unsigned char *buffer, int32 length)`
- **Purpose:** Public entry point; compute CRC32 for a memory buffer
- **Inputs:** `buffer` (byte pointer), `length` (buffer size in bytes)
- **Outputs/Return:** `uint32` CRC32 value; returns 0 if table build fails
- **Side effects:** Builds/frees CRC table; performs assertion on non-null buffer
- **Calls:** `build_crc_table()`, `calculate_buffer_crc()`, `free_crc_table()`
- **Notes:** Applies XOR pre/post-conditioning (0xFFFFFFFF) for CRC32 standard compatibility

### calculate_data_crc_ccitt
- **Signature:** `uint16 calculate_data_crc_ccitt(unsigned char *data, int32 length)`
- **Purpose:** Public entry point; compute CCITT CRC16 for a memory buffer (table-driven)
- **Inputs:** `data` (byte pointer), `length` (buffer size in bytes)
- **Outputs/Return:** `uint16` CCITT CRC16 value
- **Side effects:** None (uses pre-computed static table)
- **Calls:** None (direct table lookup loop)
- **Notes:** Uses pre-computed `crc_ccitt_table`; no dynamic allocation; standard CCITT polynomial 0x1021

### build_crc_table (private)
- **Signature:** `static bool build_crc_table(void)`
- **Purpose:** Dynamically construct the CRC32 lookup table on first use
- **Inputs:** None
- **Outputs/Return:** `bool` true (always; assert fails if table already exists)
- **Side effects:** Allocates `crc_table` array on heap; asserts `crc_table` is null on entry
- **Calls:** `new[]`
- **Notes:** Table generation uses bit-shifting and polynomial feedback loop (8 iterations per index); idempotent per invocation

### free_crc_table (private)
- **Signature:** `static void free_crc_table(void)`
- **Purpose:** Deallocate the dynamic CRC32 lookup table after use
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Deletes `crc_table` array; sets pointer to null; asserts table exists on entry
- **Calls:** `delete[]`
- **Notes:** Must be called after each CRC computation to avoid persistent allocation

### calculate_buffer_crc (private)
- **Signature:** `static uint32 calculate_buffer_crc(int32 count, uint32 crc, void *buffer)`
- **Purpose:** Incremental CRC32 computation for a single buffer chunk
- **Inputs:** `count` (bytes to process), `crc` (initial CRC state), `buffer` (data pointer)
- **Outputs/Return:** `uint32` updated CRC state
- **Side effects:** None (read-only on buffer and crc_table)
- **Calls:** None (direct table lookup and XOR operations)
- **Notes:** Core algorithm; processes one byte at a time using table lookup; portable to incremental APIs

### calculate_file_crc (private)
- **Signature:** `static uint32 calculate_file_crc(unsigned char *buffer, short buffer_size, OpenedFile& OFile)`
- **Purpose:** Read file in chunks and compute running CRC32
- **Inputs:** `buffer` (work buffer), `buffer_size` (chunk size), `OFile` (open file handle)
- **Outputs/Return:** `uint32` final CRC32; returns 0 on any I/O failure
- **Side effects:** Reads entire file sequentially; saves initial file position and restores at end; may fail if `GetPosition`, `GetLength`, `SetPosition`, or `Read` fail
- **Calls:** `OFile.GetPosition()`, `OFile.GetLength()`, `OFile.SetPosition()`, `OFile.Read()`, `calculate_buffer_crc()`
- **Notes:** Handles partial reads gracefully (takes minimum of remaining length and buffer size); applies XOR pre/post-conditioning (0xFFFFFFFF)

## Control Flow Notes
- **Initialization:** CRC32 table is built on-demand at start of each CRC32 computation and freed immediately after.
- **File Processing:** Files are processed in `BUFFER_SIZE` (1024-byte) chunks to balance memory usage and I/O efficiency.
- **Data Processing:** Memory buffers are processed byte-by-byte through a table lookup loop.
- **Shutdown:** No persistent cleanup; all allocations are scoped to individual CRC computations.
- **State Preservation:** File position is saved before reading and restored afterward to avoid side effects on callers.

## External Dependencies
- **Notable includes:** `cseries.h` (type definitions), `FileHandler.h` (FileSpecifier, OpenedFile abstractions), `crc.h` (public interface)
- **Defined elsewhere:** `FileSpecifier` class, `OpenedFile` class, `uint32` / `uint16` types, `byte` typedef, `assert` macro
- **Standard library:** `stdlib.h` (memory allocation)
