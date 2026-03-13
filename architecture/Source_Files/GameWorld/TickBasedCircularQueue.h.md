# Source_Files/GameWorld/TickBasedCircularQueue.h

## File Purpose
Header-only template library implementing tick-based circular queues for the Aleph One game engine. Provides reader-writer synchronized circular buffers keyed by game tick, supporting concurrent access by separate reader and writer threads, with options for queue duplication and mutable element access.

## Core Responsibilities
- Define abstract write-only interface (`WritableTickBasedCircularQueue`) for enforcing producer-consumer contracts
- Implement concrete circular queue with tick-indexed storage and wrap-around buffer management
- Support broadcast writes via queue duplication (`DuplicatingTickBasedCircularQueue`)
- Enable post-enqueue element mutation under careful synchronization
- Manage capacity constraints to prevent buffer overruns in concurrent context
- Handle tick wrapping via modulo arithmetic for infinite game tick sequences

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `WritableTickBasedCircularQueue<T>` | Template class (abstract interface) | Write-only interface defining reset, enqueue, availableCapacity, getWriteTick |
| `ConcreteTickBasedCircularQueue<T>` | Template class (concrete) | Full circular queue with tick-indexed buffer, reader and writer methods |
| `DuplicatingTickBasedCircularQueue<T>` | Template class (aggregator) | Broadcasts enqueue calls to multiple child `WritableTickBasedCircularQueue` instances |
| `MutableElementsTickBasedCircularQueue<T>` | Template class (subclass) | Extends ConcreteTickBasedCircularQueue; exposes `at()`/`operator[]` for post-enqueue modifications |

## Global / File-Static State
None.

## Key Functions / Methods

### ConcreteTickBasedCircularQueue::enqueue(const tValueType& inFlags)
- Signature: `void enqueue(const tValueType& inFlags)`
- Purpose: Add element to queue at current write tick
- Inputs: `inFlags` ΓÇö element to enqueue
- Outputs/Return: void
- Side effects: Increments `mWriteTick`; modifies circular buffer at write position
- Calls: `availableCapacity()` (via assert)
- Notes: Asserts capacity > 0; uses positive-tick loop before modulo to handle negative tick wrapping

### ConcreteTickBasedCircularQueue::peek(int32 inTick) const
- Signature: `const tValueType& peek(int32 inTick) const`
- Purpose: Read element at arbitrary tick without advancing read pointer
- Inputs: `inTick` ΓÇö target tick to read
- Outputs/Return: const reference to element
- Side effects: None
- Calls: `elementForTick(inTick)`
- Notes: Reader-safe; used for lookahead operations

### ConcreteTickBasedCircularQueue::dequeue()
- Signature: `void dequeue()`
- Purpose: Advance read pointer forward by one tick
- Inputs: None
- Outputs/Return: void
- Side effects: Increments `mReadTick`
- Calls: `size()` (via assert)
- Notes: Asserts queue non-empty; does not deallocate, only moves read position

### DuplicatingTickBasedCircularQueue::enqueue(const tValueType& inFlags)
- Signature: `void enqueue(const tValueType& inFlags)`
- Purpose: Broadcast enqueue to all child queues atomically
- Inputs: `inFlags` ΓÇö element to enqueue to all children
- Outputs/Return: void
- Side effects: Modifies all child queues; returns capacity of most-constrained child
- Calls: `(*i)->enqueue(inFlags)` for each child in `mChildren`
- Notes: Requires `mChildren` collection not be modified during writes; asserts non-empty

### ConcreteTickBasedCircularQueue::ConcreteTickBasedCircularQueue(const ConcreteTickBasedCircularQueue&)
- Signature: `ConcreteTickBasedCircularQueue(const ConcreteTickBasedCircularQueue<tValueType>& o)`
- Purpose: Deep copy constructor
- Inputs: `o` ΓÇö source queue
- Outputs/Return: new queue instance
- Side effects: Allocates new buffer; copies elements from `getReadTick()` to `getWriteTick()`
- Calls: `peek()`, `elementForTick()` (element copy)
- Notes: Inefficient for non-POD types; comment suggests vector or placement-new approach preferred

**Trivial accessors:** `getReadTick()`, `getWriteTick()`, `size()`, `totalCapacity()`, `availableCapacity()`, `reset()`, `at()`, `operator[]`, `children()`.

## Control Flow Notes
Used in tick-based game loop synchronization. Writer threads enqueue action flags at each game tick; reader threads asynchronously dequeue and process them. Circular buffer provides bounded memory for arbitrary tick ranges. `DuplicatingTickBasedCircularQueue` enables multi-cast broadcasts (e.g., input replayed to multiple player queues). Design enforces reader-writer safety: comment (3) guarantees `enqueue()` safe when `availableCapacity() > 0`; comment (2) guarantees `dequeue()`/`peek()` safe when queue non-empty.

## External Dependencies
- `cseries.h`: Type definitions (`int32`, `int16`), assertion macros
- `<set>`: STL `std::set` for child queue collection in `DuplicatingTickBasedCircularQueue`
- Standard C++ assert; modulo operator for buffer wrapping
