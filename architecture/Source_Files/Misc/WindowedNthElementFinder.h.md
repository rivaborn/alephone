# Source_Files/Misc/WindowedNthElementFinder.h

## File Purpose
A generic template class that maintains a sliding window of recently inserted elements and provides efficient access to the nth smallest or largest element in that window. Used for statistical queries over bounded time windows (e.g., latency percentiles, ranked metrics).

## Core Responsibilities
- Maintain a fixed-size window of elements using a circular queue
- Keep window elements sorted via an auxiliary multiset
- Provide O(n) lookups for nth smallest/largest element
- Automatically evict oldest elements when window reaches capacity
- Support resizing and resetting the window

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CircularQueue<tElementType>` | Template class (member) | Tracks insertion order and enforces window size limit |
| `std::multiset<tElementType>` | STL container (member) | Maintains elements in sorted order for percentile queries |

## Global / File-Static State
None.

## Key Functions / Methods

### WindowedNthElementFinder() [default constructor]
- Signature: `WindowedNthElementFinder() : mQueue(0) {}`
- Purpose: Initialize with zero window size
- Outputs: Empty finder object
- Notes: No elements can be inserted until `reset(windowSize)` is called

### WindowedNthElementFinder(unsigned int inWindowSize) [sized constructor]
- Signature: `explicit WindowedNthElementFinder(unsigned int inWindowSize) : mQueue(inWindowSize) {}`
- Purpose: Initialize with specified window capacity
- Inputs: `inWindowSize` ΓÇô max elements to track
- Outputs: Ready-to-use finder object

### reset()
- Signature: `void reset()` and `void reset(unsigned int inWindowSize)`
- Purpose: Clear all elements; optionally set new window size
- Inputs: (optional) `inWindowSize`
- Outputs: None
- Side effects: Clears `mSortedElements`, resets `mQueue`
- Calls: `mQueue.reset()`, `mSortedElements.clear()`

### insert(const tElementType& inNewElement)
- Signature: `void insert(const tElementType& inNewElement)`
- Purpose: Add element to window; evict oldest if at capacity
- Inputs: `inNewElement` ΓÇô element to insert
- Outputs: None
- Side effects: Updates both `mQueue` and `mSortedElements`; may deallocate oldest element
- Calls: `window_full()`, `mQueue.peek()`, `mSortedElements.erase()`, `mQueue.dequeue()`, `mSortedElements.insert()`, `mQueue.enqueue()`
- Notes: Eviction happens FIFO when window is full

### nth_smallest_element(unsigned int n)
- Signature: `const tElementType& nth_smallest_element(unsigned int n)`
- Purpose: Return the nth smallest element (0-based indexing)
- Inputs: `n` ΓÇô rank (0 = smallest)
- Outputs: Const reference to element
- Calls: `mSortedElements.begin()`, iterator increment
- Notes: Asserts `n < size()`; O(n) linear scan of multiset

### nth_largest_element(unsigned int n)
- Signature: `const tElementType& nth_largest_element(unsigned int n)`
- Purpose: Return the nth largest element (0-based indexing)
- Inputs: `n` ΓÇô rank (0 = largest)
- Outputs: Const reference to element
- Calls: `mSortedElements.rbegin()`, reverse iterator increment
- Notes: Asserts `n < size()`; O(n) linear scan in reverse

Trivial accessors: `window_full()`, `size()`, `window_size()` ΓÇô query queue state via delegated calls.

## Control Flow Notes
Not part of a traditional game loop. Used as a utility for collecting statistics over bounded windows: metrics are inserted via `insert()`, and percentile/ranked queries are answered via `nth_smallest_element()` / `nth_largest_element()`. Likely employed in profiling, network latency analysis, or gameplay event filtering.

## External Dependencies
- `CircularQueue.h` ΓÇô circular FIFO queue with fixed capacity
- `<set>` ΓÇô `std::multiset` for sorted element tracking
- `assert` (from `csalerts.h` transitively) ΓÇô runtime precondition checks
