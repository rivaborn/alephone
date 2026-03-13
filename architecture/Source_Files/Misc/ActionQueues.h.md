# Source_Files/Misc/ActionQueues.h

## File Purpose
Encapsulates multi-player action flag (input command) queues for asynchronous input handling. Manages a set of per-player circular queues to buffer player actions, with support for game state queries (zombie controllability). Part of the Marathon: Aleph One networking/input system.

## Core Responsibilities
- Allocate and manage circular queues for each player's action flags
- Enqueue player input actions (flags) with variable batch sizes
- Dequeue and peek action flags without removing them from queue
- Track queue capacity and occupancy per player
- Manage "zombies controllable" game state property as queue-set attribute
- Support in-place flag modification for specialized input processing

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ActionQueues` | class | Owns and manages a set of per-player circular queues for action flags |
| `action_queue` | struct (nested, protected) | Per-player queue metadata: read/write indices and data buffer |
| `ModifiableActionQueues` | class | Extends `ActionQueues` to allow in-place modification of queued flags |

## Global / File-Static State
None.

## Key Functions / Methods

### Constructor
- Signature: `ActionQueues(unsigned int inNumPlayers, unsigned int inQueueSize, bool inZombiesControllable)`
- Purpose: Initialize queue set for `inNumPlayers` players, each with capacity `inQueueSize-1` entries
- Inputs: Number of players, queue size per player, initial zombie controllability flag
- Outputs/Return: New queue set object
- Side effects: Allocates `mQueueHeaders` array and shared `mFlagsBuffer`
- Calls: (defined in .cpp)
- Notes: Queue size is actual capacity + 1 (one entry reserved for circular queue sentinel)

### reset / resetQueue
- Signature: `void reset()` / `void resetQueue(int inPlayerIndex)`
- Purpose: Clear all queues or specific player's queue
- Inputs: Player index (for `resetQueue`)
- Outputs/Return: None
- Side effects: Resets read/write indices
- Calls: (defined in .cpp)
- Notes: Maintains queue structure; no deallocation

### enqueueActionFlags
- Signature: `void enqueueActionFlags(int inPlayerIndex, const uint32* inFlags, int inFlagsCount)`
- Purpose: Add batch of action flags to player's queue
- Inputs: Player index, pointer to flag array, count of flags
- Outputs/Return: None
- Side effects: Advances write pointer in target queue
- Calls: (defined in .cpp)
- Notes: Supports batched input (multiple flags per enqueue)

### dequeueActionFlags
- Signature: `uint32 dequeueActionFlags(int inPlayerIndex)`
- Purpose: Remove and return next action flag from queue
- Inputs: Player index
- Outputs/Return: Next `uint32` action flag
- Side effects: Advances read pointer; queue occupancy decreases
- Calls: (defined in .cpp)
- Notes: Circular queue; undefined behavior if queue empty

### peekActionFlags
- Signature: `uint32 peekActionFlags(int inPlayerIndex, size_t inElementsFromHead)`
- Purpose: Inspect action flag at offset without removing
- Inputs: Player index, offset from read head
- Outputs/Return: `uint32` flag at that offset
- Side effects: None
- Calls: (defined in .cpp)
- Notes: Offset 0 = head; allows lookahead without mutation

### countActionFlags / totalCapacity / availableCapacity
- Signatures:  
  `unsigned int countActionFlags(int inPlayerIndex)`  
  `unsigned int totalCapacity(int inPlayerIndex)` (inline)  
  `unsigned int availableCapacity(int inPlayerIndex)` (inline)
- Purpose: Query queue depth, max capacity, and free space
- Inputs: Player index
- Outputs/Return: Queue occupancy/capacity counts
- Side effects: None
- Calls: `countActionFlags` calls (defined in .cpp); inline methods call each other
- Notes: `totalCapacity = mQueueSize - 1`; `available = total - current count`

### zombiesControllable / setZombiesControllable
- Signatures: `bool zombiesControllable()` / `void setZombiesControllable(bool inZombiesControllable)`
- Purpose: Query and set whether zombie players can be controlled (Pfhortran script feature)
- Inputs: Boolean for setter
- Outputs/Return: Boolean for getter
- Side effects: Updates `mZombiesControllable` member
- Calls: None
- Notes: Property of queue-set; affects all players simultaneously

### modifyActionFlags (ModifiableActionQueues)
- Signature: `void modifyActionFlags(int inPlayerIndex, uint32 inFlags, uint32 inFlagsMask)`
- Purpose: In-place modify flags at queue head (bitwise OR with mask)
- Inputs: Player index, flags to apply, mask indicating which bits to modify
- Outputs/Return: None
- Side effects: Mutates head flag without removing from queue
- Calls: (defined in .cpp)
- Notes: Specialized for adjusting already-queued input (e.g., adding modifier keys)

## Control Flow Notes
Likely used during game frame update: player input is enqueued on network receive or local input event, then dequeued during physics/action processing. The circular queue design and batched enqueue suggest handling of network packet boundaries. Peek and modify support debugging and input filtering without losing state.

## External Dependencies
- **Includes**: `cseries.h` (platform abstraction, basic types, SDL2 integration)
- **Types used**: `uint32`, `size_t` (standard C++ / platform headers via cseries.h)
- **Defined elsewhere**: Implementation in corresponding `.cpp` file (not provided)
