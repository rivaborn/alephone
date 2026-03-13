# Source_Files/Misc/CircularByteBuffer.cpp

## File Purpose
Implements a circular queue for byte-oriented data with support for efficient bulk operations. Provides both copy-based and zero-copy interfaces for enqueuing and reading bytes, automatically handling wraparound across buffer boundaries.

## Core Responsibilities
- Enqueue bytes into the circular buffer with automatic wraparound handling
- Peek (read) bytes from the buffer without advancing read index
- Provide zero-copy variants (`NoCopy` methods) to avoid `memcpy` overhead by exposing raw buffer pointers
- Calculate chunk boundaries when data spans the buffer wraparound point
- Support writing/reading operations split across two buffer regions (before and after wraparound)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CircularByteBuffer` | class | Inherits from `CircularQueue<char>` to implement a byte-oriented circular queue |
| `std::pair<unsigned int, unsigned int>` | struct | Encodes chunk sizes for split read/write operations (first chunk size, second chunk size) |

## Global / File-Static State
None.

## Key Functions / Methods

### splitIntoChunks
- Signature: `static std::pair<unsigned int, unsigned int> splitIntoChunks(unsigned int inByteCount, unsigned int inStartingIndex, unsigned int inQueueSize)`
- Purpose: Compute chunk boundaries when a read/write operation wraps around the buffer's circular boundary
- Inputs: byte count to process, starting index in buffer, total queue size
- Outputs/Return: pair of (first chunk size, second chunk size) where both sum to `inByteCount`
- Side effects: None
- Calls: `std::min()`
- Notes: Critical utility for handling wraparound; if no wraparound, second chunk is 0

### enqueueBytes
- Signature: `void enqueueBytes(const void* inBytes, unsigned int inByteCount)`
- Purpose: Copy data into buffer, handling wraparound via two `memcpy` calls if needed
- Inputs: source buffer pointer, byte count
- Outputs/Return: (none)
- Side effects: Advances write index via `advanceWriteIndex()`; modifies `mData`
- Calls: `splitIntoChunks()`, `memcpy()`, `advanceWriteIndex()`, `getRemainingSpace()`
- Notes: Asserts that sufficient space is available; skips copy-phase if byte count is 0

### enqueueBytesNoCopyStart
- Signature: `void enqueueBytesNoCopyStart(unsigned int inByteCount, void** outFirstBytes, unsigned int* outFirstByteCount, void** outSecondBytes, unsigned int* outSecondByteCount)`
- Purpose: Provide direct pointers into buffer for writing, avoiding copy; first half of a two-stage write
- Inputs: byte count to reserve, output pointers (may be NULL)
- Outputs/Return: Fills output pointers/counts with buffer regions; returns NULL pointers if byte count is 0
- Side effects: None (index not advanced until `enqueueBytesNoCopyFinish()`)
- Calls: `splitIntoChunks()`, `getWriteIndex()`, `getRemainingSpace()`
- Notes: Caller writes directly to returned pointers, then calls `enqueueBytesNoCopyFinish()`; allows writing fewer bytes than requested

### enqueueBytesNoCopyFinish
- Signature: `void enqueueBytesNoCopyFinish(unsigned int inActualByteCount)`
- Purpose: Complete a no-copy enqueue by advancing the write index
- Inputs: actual byte count written (Γëñ the count passed to `Start`)
- Outputs/Return: (none)
- Side effects: Advances write index via `advanceWriteIndex()`
- Calls: `advanceWriteIndex()`
- Notes: Must be paired with `enqueueBytesNoCopyStart()`; allows partial writes

### peekBytes
- Signature: `void peekBytes(void* outBytes, unsigned int inByteCount)`
- Purpose: Copy data from buffer to caller's buffer without advancing read index
- Inputs: destination buffer, byte count to read
- Outputs/Return: Fills caller's buffer with data
- Side effects: None (no index advancement)
- Calls: `splitIntoChunks()`, `memcpy()`, `getCountOfElements()`
- Notes: Asserts sufficient data is available; skips copy if byte count is 0

### peekBytesNoCopy
- Signature: `void peekBytesNoCopy(unsigned int inByteCount, const void** outFirstBytes, unsigned int* outFirstByteCount, const void** outSecondBytes, unsigned int* outSecondByteCount)`
- Purpose: Provide direct read pointers into buffer without copying; does not advance read index
- Inputs: byte count to read, output pointers (may be NULL)
- Outputs/Return: Fills output pointers/counts with buffer regions; returns NULL if byte count is 0
- Side effects: None
- Calls: `splitIntoChunks()`, `getReadIndex()`, `getCountOfElements()`
- Notes: Read index not advanced; caller may read fewer bytes than requested; allows `writev()`-style scatter operations

## Control Flow Notes
This is a utility class for I/O buffering, likely used in the networking or serialization subsystem. The NoCopy variants exist to avoid `memcpy` overhead by allowing direct buffer access (e.g., feeding buffer pointers directly to `readv()` or `writev()` syscalls). All operations defer to `CircularQueue<char>` for index management and capacity checks.

## External Dependencies
- **Base class**: `CircularQueue<char>` (defined elsewhere; provides index management, capacity queries)
- **Standard library**: `<utility>` (std::pair), `<algorithm>` (std::min), `<cstring>` (memcpy)
- **Engine**: `cseries.h` (assert macro)
