# Source_Files/Misc/ActionQueues.h - Enhanced Analysis

## Architectural Role

`ActionQueues` acts as a **buffering gateway between input capture and physics simulation** in Aleph One's real-time loop. It decouples the frame-paced input acquisition (which may receive batches from network or local hardware) from the deterministic 30 FPS physics tick that consumes actions. The per-player queue isolation is critical for **multiplayer consistency**ΓÇöeach player's action stream can be managed independently, allowing network packet boundaries and prediction rollbacks to be handled without blocking other players' processing.

The "zombies controllable" property surfaces **Pfhortran scripting semantics** into the input system itself, making script-driven AI control a first-class concern rather than a post-hoc override.

## Key Cross-References

### Incoming (who depends on this file)

- **Input subsystem** (`Source_Files/Input/joystick_sdl.cpp`, keyboard polling): Likely calls `enqueueActionFlags()` with hardware input converted to action flag bitmasks per frame or per network packet arrival.
- **GameWorld main loop** (`Source_Files/GameWorld/marathon2.cpp`, `update_world`): Expected caller of `dequeueActionFlags()` during the 30 FPS physics tick to drive player movement, weapon firing, and item interaction.
- **Network prediction subsystem** (`Source_Files/Network/`): May use `peekActionFlags()` and `availableCapacity()` to lookahead without consuming, supporting client-side prediction and rollback.
- **Pfhortran / Lua scripting** (`Source_Files/Lua/`): `zombiesControllable()` property is exposed to scripts to enable AI player control, suggesting a script-querying callback on this class.

### Outgoing (what this file depends on)

- **CSeries** (`cseries.h`): Platform abstraction for `uint32`, `size_t`, and allocation macros (not visible in header, but required by implementation).
- No other subsystem dependencies visible; this is intentionally isolated to be a pure data structure.

## Design Patterns & Rationale

**Circular Queue with Sentinel**: The `(size - 1)` effective capacity hints at a classic ring-buffer sentinel patternΓÇöone slot is reserved to distinguish full from empty (read_index == write_index indicates empty; write_index + 1 == read_index mod size indicates full). This avoids requiring a separate counter.

**Per-Player Isolation**: Separate queues per player allow **asynchronous, independent input handling**ΓÇöif one player's network packet is delayed, others are unblocked. Critical for peer-to-peer multiplayer where bandwidth/latency varies per peer.

**Batch Enqueue**: `enqueueActionFlags(int inPlayerIndex, const uint32* inFlags, int inFlagsCount)` supports stuffing multiple flags in one call. This amortizes per-enqueue overhead and aligns with **network packet boundary alignment**ΓÇöa single IP packet may contain frames worth of input; batching respects the packet structure rather than unpacking individual frames.

**Mutable Lookahead** (`ModifiableActionQueues::modifyActionFlags`): Allows in-place mutation of the head flag **without dequeuing**. Likely used for **input filtering or augmentation** (e.g., adding a modifier key retroactively, or masking out invalid actions during prediction rollback) without losing the original action from the queue.

**Zombie Controllability as Queue Property**: Instead of threading a flag through every dequeue call, it's stored once on the queue-set. Reflects the design principle that **game state controlling input interpretation is colocated with the input storage itself**ΓÇöscripts check this before processing zombie actions.

## Data Flow Through This File

```
ΓöîΓöÇ Input Events (hardware, network) ΓöÇΓöÇΓåÆ [enqueueActionFlags] ΓöÇΓöÇΓåÆ mFlagsBuffer
Γöé                                         (per-player index)       Γåô
Γöé                                                              [Circular Queue]
Γöé                                                          (read/write indices)
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
                                    Γåô
                         [dequeueActionFlags]
                         or [peekActionFlags]
                                    Γåô
                      Game Physics Tick (30 FPS)
                      + Player input processing
                      + Monster/weapon updates
```

**State Transitions per Player**:
- **Empty**: `read_index == write_index`
- **Normal**: `read_index` trails `write_index`
- **Full (sentinel)**: `write_index + 1 (mod size) == read_index`
- **Reset**: Both indices reset to 0

Capacity is fixed at construction; **overflow handling is caller's responsibility** (they must check `availableCapacity()` before enqueuing).

## Learning Notes

**Era-specific design**: This queue design reflects **early-2000s networking constraints**:
- **Deterministic tick-based physics** (30 FPS fixed) is central; async input buffering is glue, not primary flow
- **Batch I/O at packet boundaries** rather than frame-by-frame streaming (modern engines often buffer per-render-frame)
- **Per-player isolation without lock-free atomics**ΓÇöassumes single-threaded access or coarse locking at the action-processing level

**Idiomatic to Marathon**:
- The "zombies controllable" property exposes **Pfhortran scripting concerns** into the core engine, a design choice reflecting Marathon's deep modding heritage
- Circular queues with sentinels were common in C-era game engines before STL containers; this shows the C++ / object-oriented veneer over procedural-era patterns

**Modern alternatives**:
- Ring-buffer implementations now use atomic counters or lock-free CAS for thread-safe producer/consumer
- Event-driven input (callback-based) vs. pull-based queuing
- Lockable action queue (`std::queue<> + mutex`) for simpler thread safety, accepting the allocation overhead

## Potential Issues

1. **Undefined Behavior on Underflow**: `dequeueActionFlags()` has no guard against calling on an empty queue. If physics processing is faster than input supply (unlikely, but possible in testing or with network lag), dequeueing an empty queue returns uninitialized memory or garbage from a stale write_index. **Mitigation**: Caller must check `countActionFlags() > 0` or input system must guarantee fill.

2. **No Thread Safety**: Multiple callers (input thread, main game loop, prediction rollback) might race on read/write indices without visible synchronization. The header doesn't show locks; **likely assumes single-threaded access or that the .cpp implementation uses a mutex or spinlock**. **Risk**: Data races on concurrent enqueue + dequeue.

3. **Zombie Controllability is Global per Queue-Set**: Setting `mZombiesControllable` affects all players simultaneously. If one player is script-controlled and another is human, toggling the flag blocks human players' input too. **Mitigation**: Likely scripts check this per-entity before processing, not relying on the queue to enforce it; the flag is more of a hint than a hard gate.

4. **No Aging or Time-Bound Expiry**: Queued actions persist until dequeued, even if stale (e.g., a network retransmit arrives after prediction rollback). **Modern engines** often attach frame timestamps and discard expired actions; here it's caller's responsibility to reset or sanitize on rollback.
