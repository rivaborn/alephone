# Source_Files/Misc/ActionQueues.cpp

## File Purpose
Implements a circular-buffer queue system for managing player action flags (input commands) in the Marathon game engine. Encapsulates per-player action queues with support for zombie player controllability, peeking, and in-place modification of queued actions.

## Core Responsibilities
- Allocate and manage circular buffers for multiple players' action queues
- Enqueue action flags from input sources (network, local input, Pfhortran scripts)
- Dequeue and consume action flags during game updates
- Peek at queued actions without consuming them
- Count available actions and detect queue overflow/underflow
- Control whether zombie (dead) players can receive queued actions
- Modify action flags at queue head (e.g., for hotkey decoding)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `action_queue` | struct | Per-player queue header; holds `read_index`, `write_index`, and `*buffer` pointer |
| `ActionQueues` | class | Manages a set of queues for all players; encapsulates memory allocation and queue operations |
| `ModifiableActionQueues` | class | Extends `ActionQueues`; adds ability to modify flags at queue head |

## Global / File-Static State
None.

## Key Functions / Methods

### Constructor
- Signature: `ActionQueues(unsigned int inNumPlayers, unsigned int inQueueSize, bool inZombiesControllable)`
- Purpose: Initialize queue set; allocate buffers and set up queue headers for all players
- Inputs: Number of players, queue size (buffer diameter), zombie controllability flag
- Outputs/Return: None (constructor)
- Side effects: Allocates `mQueueHeaders` array and `mFlagsBuffer` contiguous block; stores configuration
- Calls: `new` for memory allocation
- Notes: Queue size is typically `ACTION_QUEUE_BUFFER_DIAMETER` (0x400); memory layout is all queue headers followed by all buffers with pointer arithmetic to distribute buffer regions

### Destructor
- Signature: `~ActionQueues()`
- Purpose: Free allocated queue buffers and headers
- Inputs: None
- Outputs/Return: None
- Side effects: Deletes `mFlagsBuffer` and `mQueueHeaders` arrays
- Calls: `delete[]`
- Notes: Checks for null pointers before deletion

### enqueueActionFlags
- Signature: `void enqueueActionFlags(int player_index, const uint32 *action_flags, int count)`
- Purpose: Add action flags to a player's queue
- Inputs: Player index, array of action flags, count of flags to queue
- Outputs/Return: None
- Side effects: Advances `write_index` (with wraparound); logs error if queue overflows; skips queueing if player is a non-controllable zombie
- Calls: `get_player_data()`, `PLAYER_IS_ZOMBIE()` macro, `logError()`
- Notes: Zombie check respects `mZombiesControllable` flag; overflow is detected when `write_index` collides with `read_index`

### dequeueActionFlags
- Signature: `uint32 dequeueActionFlags(int player_index)`
- Purpose: Remove and return next action flag from a player's queue
- Inputs: Player index
- Outputs/Return: Single `uint32` action flag (0 if empty or zombie)
- Side effects: Advances `read_index`; logs error if queue is empty or on zombie underflow
- Calls: `get_player_data()`, `PLAYER_IS_ZOMBIE()` macro, `logError()`
- Notes: Non-controllable zombies always return 0; empty queues also return 0 with error log

### peekActionFlags
- Signature: `uint32 peekActionFlags(int inPlayerIndex, size_t inElementsFromHead)`
- Purpose: Examine an action flag at a given offset without consuming it
- Inputs: Player index, offset from queue head (0 = next to dequeue)
- Outputs/Return: Action flag at that offset (0 if out of bounds or zombie)
- Side effects: None (read-only)
- Calls: `get_player_data()`, `PLAYER_IS_ZOMBIE()` macro, `countActionFlags()`, `logError()`
- Notes: Logs error if peeking beyond available flags; uses modulo arithmetic to handle circular wraparound

### countActionFlags
- Signature: `unsigned int countActionFlags(int player_index)`
- Purpose: Return number of queued action flags available for a player
- Inputs: Player index
- Outputs/Return: Count of available flags (unsigned int)
- Side effects: None
- Calls: `get_player_data()`, `PLAYER_IS_ZOMBIE()` macro
- Notes: Non-controllable zombies report queue size (mQueueSize) to allow infinite reads; uses modulo arithmetic `(mQueueSize + write_index - read_index) % mQueueSize`

### modifyActionFlags (ModifiableActionQueues)
- Signature: `void modifyActionFlags(int inPlayerIndex, uint32 inFlags, uint32 inFlagsMask)`
- Purpose: Modify action flags at the queue head (next to dequeue)
- Inputs: Player index, new flag bits, bitmask of which bits to modify
- Outputs/Return: None
- Side effects: Modifies `queue->buffer[queue->read_index]` in-place if non-zero and queue not empty
- Calls: `countActionFlags()`, `logError()`
- Notes: Logs error if queue is empty; skips modification if flags are 0xffffffff (sentinel value); bitwise operation applies mask selectively

## Control Flow Notes
This is a utility class consumed by the game update loop (visible in player.h's `update_players()` signature taking `ActionQueues*`). Action flags flow: input ΓåÆ enqueue ΓåÆ update loop ΓåÆ dequeue/peek ΓåÆ player physics update. Circular queue prevents allocation churn. Zombie behavior is configurable for Pfhortran scripting vs. normal gameplay.

## External Dependencies
- **player.h**: `player_data` struct, `get_player_data(int)`, `PLAYER_IS_ZOMBIE(p)` macro
- **Logging.h**: `logError(...)` macro for overflow/underflow diagnostics
