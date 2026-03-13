# Source_Files/Misc/CircularByteBuffer.h

## File Purpose
A circular queue specialization for byte-level data with support for efficient bulk operations and zero-copy access patterns. Handles wraparound complexity automatically when data chunks cross the buffer's logical boundary.

## Core Responsibilities
- Provide a typed circular queue of bytes (`char`) with convenient bulk enqueue/peek semantics
- Support copy-based chunk operations (peekBytes, enqueueBytes) with automatic wraparound
- Expose zero-copy direct buffer pointers for performance-critical I/O (writev/readv style)
- Offer split-point calculations to decompose operations across buffer wraparound
- Manage the two-phase no-copy enqueue protocol (start ΓåÆ write ΓåÆ finish)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| CircularByteBuffer | class | Public interface; inherits from `CircularQueue<char>` to specialize for byte buffering |
| CircularByteBufferBase | typedef | Alias: `CircularQueue<char>` |

## Global / File-Static State
None.

## Key Functions / Methods

### peekBytes
- Signature: `void peekBytes(void* outBytes, unsigned int inByteCount)`
- Purpose: Read a contiguous chunk of bytes from the front of the queue without removing them
- Inputs: destination buffer pointer, byte count to read
- Outputs/Return: fills `outBytes` with data; none returned
- Side effects: read index unchanged; accesses queue memory
- Calls: (implementation in .cpp, not visible here)
- Notes: Caller must verify `inByteCount <= getCountOfElements()` before calling; handles wraparound internally

### enqueueBytes
- Signature: `void enqueueBytes(const void* inBytes, unsigned int inByteCount)`
- Purpose: Append a contiguous chunk of bytes to the queue's write end
- Inputs: source buffer pointer, byte count to write
- Outputs/Return: none
- Side effects: advances write index; modifies queue data
- Calls: (implementation in .cpp)
- Notes: Caller must verify `inByteCount <= getRemainingSpace()` before calling; handles wraparound internally

### peekBytesNoCopy
- Signature: `void peekBytesNoCopy(unsigned int inByteCount, const void** outFirstBytes, unsigned int* outFirstByteCount, const void** outSecondBytes, unsigned int* outSecondByteCount)`
- Purpose: Provide direct pointers into queue memory for reading, avoiding a copy when data may span the wraparound boundary
- Inputs: byte count to read
- Outputs/Return: four optional output parameters: first chunk pointer/length, second chunk pointer/length (null if unneeded)
- Side effects: read index **not advanced** (peek-only); no data modification
- Calls: (implementation in .cpp)
- Notes: Caller must verify `inByteCount <= getCountOfElements()`; read index advances only on explicit `dequeue()` call; suitable for writev-style operations

### enqueueBytesNoCopyStart
- Signature: `void enqueueBytesNoCopyStart(unsigned int inByteCount, void** outFirstBytes, unsigned int* outFirstByteCount, void** outSecondBytes, unsigned int* outSecondByteCount)`
- Purpose: Begin a zero-copy enqueue by providing writable pointers into the buffer at the write position
- Inputs: intended byte count to write
- Outputs/Return: four optional output parameters: first writable chunk pointer/length, second chunk pointer/length
- Side effects: **write index not advanced** (caller must call `enqueueBytesNoCopyFinish`)
- Calls: (implementation in .cpp)
- Notes: Caller must verify `inByteCount <= getRemainingSpace()`; must be paired with `enqueueBytesNoCopyFinish()` after data is written; caller may write fewer bytes than requested, reported at finish

### enqueueBytesNoCopyFinish
- Signature: `void enqueueBytesNoCopyFinish(unsigned int inActualByteCount)`
- Purpose: Finalize a zero-copy enqueue by advancing the write index based on actual bytes written
- Inputs: count of bytes actually written into the buffers from `enqueueBytesNoCopyStart`
- Outputs/Return: none
- Side effects: advances write index by `inActualByteCount`
- Calls: (implementation in .cpp)
- Notes: Separating start/finish ensures data is written before index moves (invariant safety); actual count can be Γëñ intended count from start call

### splitIntoChunks
- Signature: `static std::pair<unsigned int, unsigned int> splitIntoChunks(unsigned int inByteCount, unsigned int inStartingIndex, unsigned int inQueueSize)`
- Purpose: Calculate how to decompose a byte count into two contiguous chunks accounting for wraparound
- Inputs: byte count needed, starting position, total queue size
- Outputs/Return: `std::pair<unsigned int, unsigned int>` where first is primary chunk size, second is remainder chunk size (0 if no wrap)
- Side effects: none (pure function)
- Calls: none visible
- Notes: Static utility available outside the class; sum of returned pair always equals `inByteCount` unless wraparound is needed; second element is 0 if no split required

## Control Flow Notes
This is a utility/data-structure header, not part of the main render/frame loop. Likely used by networking, I/O, or buffering subsystems that need efficient serialization/deserialization of byte streams.

## External Dependencies
- `#include <utility>` for `std::pair`
- `#include "CircularQueue.h"` for base template class `CircularQueue<T>` (provides core ring-buffer management: read/write indices, storage)
- Implicitly relies on `new`/`delete` for dynamic allocation (via CircularQueue)
