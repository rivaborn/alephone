# Source_Files/Misc/CircularQueue.h

## File Purpose
Template-based circular queue (ring buffer) implementation for the Marathon: Aleph One game engine. Provides fixed-capacity FIFO container with efficient O(1) enqueue/dequeue operations using modular index wrapping. Supports copy construction and assignment since May 2003.

## Core Responsibilities
- Manage a dynamically allocated circular buffer with configurable capacity
- Maintain read and write indices with automatic wrap-around to implement FIFO ordering
- Provide enqueue (append), dequeue (remove), and peek operations
- Track queue occupancy and available space
- Handle buffer reallocation on size reset
- Enforce bounds with debug assertions (capacity check on both indices)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CircularQueue<T>` | Template class | Generic circular buffer; T is the element type |

## Global / File-Static State
None.

## Key Functions / Methods

### `CircularQueue()` (default constructor)
- Signature: `CircularQueue()`
- Purpose: Construct empty queue with zero capacity
- Inputs: None
- Outputs/Return: N/A (constructor)
- Side effects: Allocates nothing; initializes `mData` to NULL and calls `reset(0)`
- Calls: `reset(0)`
- Notes: Queue is unusable until `reset(size)` is called

### `CircularQueue(unsigned int inSize)` (sized constructor)
- Signature: `CircularQueue(unsigned int inSize)`
- Purpose: Construct queue with requested capacity
- Inputs: `inSize` ΓÇö number of elements the queue can hold
- Outputs/Return: N/A (constructor)
- Side effects: Allocates `inSize + 1` elements on heap
- Calls: `reset(inSize)`
- Notes: Allocates one extra slot to distinguish empty from full state

### `operator=(const CircularQueue<T>&)` (copy assignment)
- Signature: `CircularQueue<T>& operator =(const CircularQueue<T>& o)`
- Purpose: Replace this queue with a copy of another
- Inputs: `o` ΓÇö source queue
- Outputs/Return: Reference to `*this`
- Side effects: Deallocates old buffer; reallocates to match source size; enqueues all source elements in order
- Calls: `reset(o.getTotalSpace())`, `getCountOfElements()`, `getReadIndex()`, `enqueue()`
- Notes: Shallow copy of data; source queue must not change during copy; self-assignment guard included

### `reset(unsigned int inSize)`
- Signature: `void reset(unsigned int inSize)`
- Purpose: Clear queue and reallocate buffer to new size
- Inputs: `inSize` ΓÇö desired capacity
- Outputs/Return: None
- Side effects: Deallocates old buffer (if any); allocates new buffer of size `inSize + 1`; resets read/write indices to 0
- Calls: (none; direct allocation via `new`/`delete`)
- Notes: Asserts wrap-around safety (`inSize + 1 > inSize`); reuses allocation if size unchanged

### `enqueue(const T& inData)`
- Signature: `void enqueue(const T& inData)`
- Purpose: Append element to queue tail
- Inputs: `inData` ΓÇö element to add
- Outputs/Return: None
- Side effects: Writes to `mData[mWriteIndex]`; advances write index by 1 (wraps)
- Calls: `getWriteIndex()`, `advanceWriteIndex()`
- Notes: Asserts remaining space > 0; fails silently if full (buffer overflow)

### `dequeue(unsigned int inAmount = 1)`
- Signature: `void dequeue(unsigned int inAmount = 1)`
- Purpose: Remove `inAmount` elements from queue head
- Inputs: `inAmount` ΓÇö number of elements to skip (default 1)
- Outputs/Return: None
- Side effects: Advances read index by `inAmount` (wraps)
- Calls: `advanceReadIndex(inAmount)`
- Notes: Asserts element count >= inAmount; no actual data deletion (index advancement only)

### `peek() const`
- Signature: `const T& peek() const`
- Purpose: Access front element without removal
- Inputs: None
- Outputs/Return: Const reference to element at read index
- Side effects: None
- Calls: `getReadIndex()`
- Notes: Asserts queue is not empty

### Query methods
- `getCountOfElements() const` ΓÇö returns number of enqueued elements (modulo arithmetic)
- `getRemainingSpace() const` ΓÇö returns `getTotalSpace() - getCountOfElements()`
- `getTotalSpace() const` ΓÇö returns max capacity (`mQueueSize - 1`)
- Notes: All three return 0 if queue is uninitialized (`mQueueSize == 0`)

## Control Flow Notes
This is a data structure utility, not part of the frame/update/render loop. Used by other game engine systems (likely event dispatch, network message buffering, or command queues) that need bounded FIFO storage. Initialization happens when engine subsystems create queue instances; no shutdown contract.

## External Dependencies
- `csalerts.h` ΓÇö for `assert()` macro (compiles to no-op in non-DEBUG builds; fatal assertion in DEBUG)
- Standard C++ runtime ΓÇö `new`/`delete` for heap allocation
