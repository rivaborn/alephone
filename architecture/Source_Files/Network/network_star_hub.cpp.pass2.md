# Source_Files/Network/network_star_hub.cpp - Enhanced Analysis

## Architectural Role

This file implements the **traffic aggregation and synchronization hub** for Aleph One's star-topology multiplayer protocol. The hub is the central authority that collects action flags from all players (spokes), synthesizes missing data via bandwidth reduction, calculates per-player timing offsets, and broadcasts synchronized game state back to all spokes. It's designed for **lossy, NAT-friendly networking** where spokes may drop packets, operate behind firewalls, and experience unpredictable jitterΓÇöenabling playable multiplayer over commodity internet.

The hub enforces **tick-based lockstep execution**: all game ticks are numbered sequentially, all player actions are keued by tick, and the hub ensures all players see the same action sequence for each tick (via synthesis for missing data). This deterministic architecture allows the game engine to run identically on all clients once initial state is synced.

## Key Cross-References

### Incoming (who depends on this file)

- **`network.cpp` / Network layer**: Dispatches incoming UDP packets to `hub_received_network_packet()` after CRC validation
- **`network_star_spoke.cpp` (local spoke)**: Calls `send_frame_to_local_spoke()` to inject packets directly into hub; receives responses via `spoke_received_network_packet()` callback
- **`network_star.h` (public interface)**: Exports `hub_initialize()`, `hub_cleanup()`, `hub_received_network_packet()`, `hub_tick()` entrypoints
- **Game loop / `marathon2.cpp`**: Implicitly depends on hub tick task running at ~30 Hz to detect net-dead players and trigger sends
- **Metaserver integration**: Game reports lag statistics from `sNetworkPlayers[*].mStats` for matchmaking/ranking

### Outgoing (what this file depends on)

- **`NetDDPSendFrame()`**: Transmits serialized game state packets to remote spokes over UDP
- **`spoke_received_network_packet()`**: Direct function call into local spoke (when not standalone), holding shared `mytm_mutex`
- **`myXTMSetup()`, `myTMRemove()`, `myTMCleanup()`** (`mytm.h`): Background task scheduling at ~30 Hz; mutex-protected concurrency
- **`TickBasedCircularQueue` + `MutableElementsTickBasedCircularQueue`**: Bounded tick-indexed queues for action flags and ACK tracking
- **`WindowedNthElementFinder<int32>`** (`WindowedNthElementFinder.h`): Calculates median arrival offset for timing adjustment decisions
- **`calculate_data_crc_ccitt()`** (`crc.h`): Validates packet integrity; discards corrupted packets silently
- **`local_random()`** (`world.h`): Seeds flag synthesis for lagging players (motion vector midpoints, etc.)
- **`InfoTree.h`** (only in standalone debug mode): Exports timing adjustment telemetry for post-mortem analysis

---

## Design Patterns & Rationale

### 1. **Tick-Based Lockstep Synchronization**
All game state is indexed by **tick number**, not wall-clock time. The hub and spokes maintain independent clocks (`sNetworkTicker` on hub, local tick counter on spoke) and periodically adjust one another via feedback messages. This enables:
- **Deterministic playback**: if both players start with the same initial state and see the same action sequence, their game state diverges only if bugs exist
- **Replay recording**: storing action flags lets deterministic engines replay exact gameplay
- **Lag tolerance**: players with different network latencies still execute synchronously by buffering and time-shifting

**Why this pattern?** Real-time game engines (Doom, Marathon, Quake) adopted lockstep before internet multiplayer was common, and it remains the gold standard for action games needing sub-frame-latency responsiveness without client-side prediction artifacts.

### 2. **Circular Queue Discipline with Completion Tracking**
The hub maintains **multiple interlocked circular queues**, all keyed by tick:
- `sFlagsQueues[player]` ΓåÉ one queue per player, action flags
- `sLateFlagsQueues[player]` ΓåÉ separate queue for late-arriving flags (after tick nominally "closes")
- `sPlayerDataDisposition` ΓåÉ bitmask per tick: which players still owe data
- `sPlayerReflectedFlags` ΓåÉ bitmask per tick: which players had synthesized flags reflected back

**Rationale**: 
- Bounded memory: all queues are fixed-size (5 seconds of ticks), no unbounded growth
- Atomic completion: a tick completes when `sPlayerDataDisposition[tick]` reaches zero (all players ACKed)
- In-place updates: `MutableElementsTickBasedCircularQueue` allows masking off bits without dequeue/requeue overhead
- **Key insight**: tick indices create a natural ordering; bitmasks enable O(1) completion detection instead of scanning linked lists

### 3. **Bandwidth Reduction via Intelligent Flag Synthesis**
Instead of **dropping data for lagging players** (causing input loss), the hub **synthesizes plausible flags** in `make_up_flags_for_first_incomplete_tick()`:
- For lagging players, collapse recent late-arriving flags to a **midpoint** (average motion vector)
- Preserve **trigger bits** (weapon fire, jump action) from late flags
- Reflect synthesized flags back to the sender so they see their own motion (even if delayed)

**Why?** Real players notice input loss immediately, but smoothed/extrapolated input is forgivable. This is similar to **lossy video codecs**ΓÇödrop non-critical data to meet bandwidth budgets, but preserve enough to maintain perceived quality.

### 4. **Feedback-Based Timing Adjustment**
The hub doesn't directly command spoke clocks. Instead, it **observes arrival patterns** and requests adjustments:
1. Hub collects per-player **arrival offset** samples (when flags arrive relative to `sNetworkTicker`)
2. Hub uses `WindowedNthElementFinder<int32>` to compute **windowed median** (robust to outliers)
3. Hub sends "adjust clock by +X ticks" message, **sticketted** until the spoke ACKs past the message tick
4. Spoke applies adjustment, resumes with new clock offset
5. Hub clears averaging window and waits for fresh data before next adjustment

**Why this pattern?** 
- **Feedback control**: avoids explicit "lock to hub clock" which would require all spokes to agree on a global reference
- **Decentralized**: each spoke can choose its own clock rate (useful for non-game applications like AI bots)
- **Robust to jitter**: windowed median filters transient packet delays
- **Sticketted messaging**: ensures delivery even if packets drop; receiver doesn't need to ACK the adjustment itself

### 5. **Hierarchical Packet Loss Mitigation**
The hub sends at different rates depending on network conditions:
- **Normal mode** (`mSendPeriod = 1`): send every tick
- **Recovery mode** (`mRecoverySendPeriod = 0.5s`): every 0.5 seconds, resend last 4 seconds of historical flags
- **Bandwidth-reduction mode**: synthesize missing flags to reduce tick count sent

**Rationale**: Recovery sends act as a **repair mechanism** if many packets drop. Every 0.5 seconds, the hub essentially says "here's the last 4 seconds of complete game state"ΓÇöa spoke can catch up from a packet burst window rather than slowly resyncing tick-by-tick.

### 6. **Local Spoke Communication via Direct Calls**
On non-standalone hubs, the local spoke and hub **do not use network packets**. Instead:
- Hub stores outgoing packet in `sLocalOutgoingBuffer`
- Hub calls `spoke_received_network_packet()` directly, holding `mytm_mutex`
- Spoke processes packet synchronously, may generate a response
- Hub processes response packet, all within the same mutex hold

**Why?** Eliminates UDP socket overhead for local communication; enables tightly-coupled timing (local spoke tick can invoke hub tick, or vice versa). The single-socket constraint of the network layer is worked around via this "loopback" mechanism.

---

## Data Flow Through This File

### Initialization
```
hub_initialize()
  Γö£ΓöÇ allocate sFlagsQueues[N] and sLateFlagsQueues[N]
  Γö£ΓöÇ initialize sNetworkPlayers[N] with connectivity state
  Γö£ΓöÇ build sAddressToPlayerIndex map (empty initially, populated on first packet)
  ΓööΓöÇ spawn background task via myXTMSetup() ΓåÆ hub_tick() every ~33ms
```

### Per-Packet Ingestion
```
hub_received_network_packet(UDPpacket)
  Γö£ΓöÇ validate CRC; discard on failure
  Γö£ΓöÇ if identification packet ΓåÆ discover player address, map in sAddressToPlayerIndex
  Γö£ΓöÇ if game data packet ΓåÆ hub_received_game_data_packet_v1()
  Γöé   Γö£ΓöÇ player_acknowledged_up_to_tick() ΓåÆ dequeue completed ticks, update latency
  Γöé   Γö£ΓöÇ process_messages() ΓåÆ handle timing adjustment requests, net-dead notifications
  Γöé   Γö£ΓöÇ player_provided_flags_from_tick_to_tick() ΓåÆ enqueue flags, track completion
  Γöé   ΓööΓöÇ conditionally send_packets() if new ticks completed
  ΓööΓöÇ check_send_packet_to_spoke() ΓåÆ route to local spoke if needed
```

### Per-Tick Background Work
```
hub_tick() [background task, ~30 Hz]
  Γö£ΓöÇ increment sNetworkTicker
  Γö£ΓöÇ for each connected player:
  Γöé   Γö£ΓöÇ if silent > timeout ΓåÆ make_player_netdead()
  Γöé   ΓööΓöÇ if lagging ΓåÆ make_up_flags_for_first_incomplete_tick() [synthesize]
  Γö£ΓöÇ optionally send_packets() [recovery send or bandwidth reduction]
  ΓööΓöÇ update per-player latency/jitter stats
```

### Packet Transmission
```
send_packets()
  Γö£ΓöÇ record sNetworkTicker in sFlagSendTimeQueue (for latency measurement)
  Γö£ΓöÇ for each connected player with known address:
  Γöé   Γö£ΓöÇ encode ACK tick range
  Γöé   Γö£ΓöÇ if outstanding timing adjustment ΓåÆ include adjustment message
  Γöé   Γö£ΓöÇ if net-dead tick set ΓåÆ include net-dead notifications
  Γöé   Γö£ΓöÇ encode action flags in **tick-major order** (all players' flags for each tick)
  Γöé   Γö£ΓöÇ if player received synthesized flags ΓåÆ include reflected flags
  Γöé   ΓööΓöÇ NetDDPSendFrame() or send_frame_to_local_spoke()
  ΓööΓöÇ update sLastNetworkTickSent and sSmallestUnsentTick
```

### Completion and Cleanup
```
hub_cleanup(graceful, postGameTick)
  Γö£ΓöÇ if graceful: hub_check_for_completion() [wait for all players to ACK postgame tick]
  Γö£ΓöÇ myTMRemove() [stop background task]
  Γö£ΓöÇ myTMCleanup() [wait for task to finish]
  ΓööΓöÇ clear all queues and player state
```

---

## Learning Notes

**For developers studying this engine:**

1. **Tick-based lockstep is counterintuitive but powerful**: Modern games often use client-side prediction + server reconciliation. Marathon predates that architecture and uses deterministic lockstep instead. It trades bandwidth (more frequent state syncs) for simplicity (no rollback/interpolation logic).

2. **Circular queues are the backbone**: This file uses them extensively. They enable O(1) memory usage, O(1) completion detection, and clean separation of "incomplete" vs. "sent-but-unacked" regions. Study `TickBasedCircularQueue` if you want to understand bounded-queue patterns.

3. **Bitmasks for tracking**: Rather than lists or maps, the hub uses **bitmasks indexed by player index** to track who-owes-what. This is memory-efficient and enables atomic operations ("all players checked in" = all bits zero).

4. **Bandwidth reduction is a design choice, not a bug fix**: The hub doesn't *synthesize* flags as a fallback; it actively chooses to when network is congested. This is a deliberate tradeoff: accept input latency/extrapolation to keep bandwidth low.

5. **Timing adjustment feedback loop**: This is probably the most subtle part. Understanding the windowed Nth-element finder + sticketted messaging + adjustment window clearing is crucial to extending this for other network scenarios.

6. **Comments document evolution**: The file's header comments trace the evolution from simple `hub_initialize(addresses)` to NAT-friendly dynamic address discovery (Sept 2004). This shows how network code evolves as deployments reveal new constraints.

---

## Potential Issues

1. **Flag synthesis quality**: For very laggy players (>2 seconds), synthesized flags diverge sharply from actual input. Players may see themselves "teleport" when synthesis finally stops and real flags arrive. Mitigation: synthesis is only applied within `mInGameWindowSize` (5 seconds), so divergence is bounded.

2. **Address ambiguity behind NAT**: Multiple players behind the same NAT gateway appear as the same IP:port pair. The code assumes each (IP:port) tuple maps to one player, but NAT port remapping violates this. Mitigation: reliance on identification packets with player IDs (line 1047), not topology-provided addresses.

3. **Timing adjustment oscillation**: If a spoke ignores adjustment messages or applies them erratically, the hub's feedback loop may oscillate. The code mitigates this by only requesting new adjustments after the averaging window refills (after previous adjustment is ACKed).

4. **Standalone mode edge case**: In standalone hub mode, if no players connect before initialization completes, `sReferencePlayerIndex` defaults to 0 (line 613). If that player is then disconnected, latency calculations may reference a dead player. Low impact since standalone hubs are not for production.

5. **Integer overflow in latency rolling sum**: `mLatencyTicks` accumulates per-tick latency samples; if buffers persist across session boundaries, could overflow `int32`. However, buffers are reset per game session, so not a practical issue.
