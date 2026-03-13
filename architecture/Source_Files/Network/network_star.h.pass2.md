# Source_Files/Network/network_star.h - Enhanced Analysis

## Architectural Role

This header defines the **public API boundary for Aleph One's peer-to-peer multiplayer networking layer**, implementing a classic hub-spoke (server-client) topology. It mediates between the game loop (which reads/writes action flags at tick boundaries) and the UDP network stack (which handles packet transmission). All multiplayer synchronization flows through the tick-based queue abstraction declared here, making this file the critical contract between deterministic game simulation and asynchronous network I/O.

## Key Cross-References

### Incoming (who depends on this file)
- **GameWorld (marathon2.cpp, player.cpp, etc.)**: Calls `spoke_initialize()` at level load; queries `spoke_get_net_time()` each tick to index into action queues; calls `spoke_check_world_update()` to determine if rendering/recomputation needed; reads `spoke_get_unconfirmed_flags_queue()` to retrieve pending player inputs
- **Network event loop / UDP dispatcher**: Calls `hub_received_network_packet()` and `spoke_received_network_packet()` on each inbound UDP datagram
- **Preferences system (XML/InfoTree)**: Calls `DefaultHubPreferences()`, `DefaultSpokePreferences()`, and parse functions to load/validate network configuration
- **Main shell/interface layer**: Calls `hub_initialize()` to start a server; `spoke_initialize()` to join; `hub_cleanup()`/`spoke_cleanup()` to disconnect gracefully

### Outgoing (what this file depends on)
- **TickBasedCircularQueue.h**: Provides `ConcreteTickBasedCircularQueue<T>` (read-only) and `WritableTickBasedCircularQueue<T>` (write-only) templates for buffering action flags by tick
- **ActionQueues.h**: Included but abstractions delegated to TickBasedCircularQueue
- **NetworkInterface.h**: Depends on `IPaddress` struct and `UDPpacket` definition for socket-level primitives
- **map.h**: Reads `TICKS_PER_SECOND` constant (30 ticks/sec) for synchronization timing; used in `kPregameTicks` calculation
- **Preferences/InfoTree system**: Hub and spoke register preference trees; parsed at startup to configure network behavior (presumably: timeouts, packet rates, bandwidth caps)

## Design Patterns & Rationale

**1. Tick-Based Synchronization**  
All multiplayer state advances in lockstep at exactly 30 ticks/second. This is *fundamental* to Mars determinism: two clients seeing identical inputs in the same tick order will simulate identically, enabling replay and verification. Modern engines often use a higher tick rate (60+) or timestamp-based replication, but 30 ticks matches the core game loop's update frequency.

**2. Circular Queue Keyed by Tick**  
Rather than a linear packet buffer, each player has a `TickBasedActionQueue` indexed by tick number. This decouples packet arrival order from logical action orderingΓÇöif packet for tick 10 arrives before packet for tick 8, the queue still reconstructs the correct sequence.

**3. Magic-Byte Packet Framing**  
Packet types identified by two-byte magic numbers (`0x5331` = 'S1', `0x4831` = 'H1', etc.) plus a 2-byte CRC header. This is era-appropriate (pre-2005) but lacks modern versioning: the protocol is monolithic. If action flags expand from 4 to 8 bytes, all clients must update atomically.

**4. Graceful vs. Forced Shutdown**  
`hub_cleanup(bool inGraceful, ...)` allows server to notify all clients *before* closing sockets. Spokes receiving a graceful shutdown packet can display "game ended by host" rather than timing out. Forced cleanup drops immediatelyΓÇöuseful if the server crashes.

**5. Local Hub Optimization**  
`spoke_initialize(..., bool inHubIsLocal)` flag. On a single machine, the spoke can write action queues directly without UDP, bypassing network latency. This is used for split-screen or testing without running two processes. Without this flag, even local hub-spoke communication incurs UDP round-trip.

**6. Preference Tree Integration**  
Both hub and spoke expose `DefaultXPreferences()`, `XPreferencesTree()`, and `XParsePreferencesTree()` functions, matching the engine's centralized XML configuration system (seen in `Source_Files/XML/`). This decouples network parameters from hardcoded constants, allowing mods to tune timeouts, packet rates, etc.

## Data Flow Through This File

**Server-Side (Hub) Flow:**
```
inPlayerAddresses[] (initialization)
    Γåô
hub_initialize() [allocates action queue state, opens UDP listen socket]
    Γåô
hub_received_network_packet(UDPpacket from spoke)
    Γåô
Extract action flags (magic 'S1'), validate CRC, buffer by tick
    Γåô
Broadcast to all other spokes (magic 'H1' or 'F1' with spoke echo-back)
    Γåô
Spokes receive, fill their action queues, game loop consumes
```

**Client-Side (Spoke) Flow:**
```
inHubAddress, inFirstTick, WritableTickBasedActionQueue[] (initialization)
    Γåô
spoke_initialize() [establish UDP connection to hub, sync ticks via 'TA' timing packets]
    Γåô
[Game loop] spoke_get_net_time() ΓåÆ current tick
    Γåô
[Game loop] read spoke_get_unconfirmed_flags_queue() ΓåÆ all players' action flags for this tick
    Γåô
[Network thread] spoke_received_network_packet(UDPpacket from hub)
    Γåô
Extract action flags, fill writable queues by tick, set spoke_check_world_update() flag
    Γåô
[Game loop] calls spoke_check_world_update() ΓåÆ true = re-render/resimulate
```

**Key State Transitions:**
- **Pregame (0 to kPregameTicks)**: Timing adjustment packets ('TA') exchanged, no real game data. Both sides synchronize clock skew and measure RTT.
- **Running**: Spokes send action flags to hub each tick; hub broadcasts back; queues are consumed by game simulation.
- **End-of-Message**: Marker packet ('EM') signals frame boundary, allowing batching of multiple ticks into one UDP datagram.
- **Disconnection**: Graceful flag signals end; forced = timeout.

## Learning Notes

**What This File Teaches:**
- **Tick-based networking is simpler than continuous-time**: You trade CPU cycles for architectural clarity. No need for complex interpolation or state predictionΓÇögame ticks are your contract.
- **Circular queues by tick are elegant for unordered packet delivery**: The queue structure itself solves packet reordering without explicit ACK/NACK.
- **Idiomatic 2003 multiplayer design**: Magic bytes, CRC checksums, fixed packet structures. Modern protocols (gRPC, Cap'n Proto, Protocol Buffers) provide versioning and schema evolution; this design assumes monolithic protocol.
- **Preferences as a system-wide pattern**: Rather than passing parameters through function calls, the engine uses a tree-based preference system. See `DefaultHubPreferences()` ΓåÆ `HubPreferencesTree()` ΓåÆ `HubParsePreferencesTree()`. This mirrors the `Source_Files/XML/` subsystem's philosophy.

**Era-Specific Design Choices:**
- No cryptographic authentication or packet signing (trust assumed on LAN or authorized servers).
- No explicit version negotiation (protocol is versioned implicitly by magic bytes, not negotiated).
- No bandwidth adaptation (packet rate is fixed; no congestion control).
- Tick-based (not frame-rate-independent): if one client lags, all clients stall. Modern games often decouple tick rate from render rate.

## Potential Issues

1. **Monolithic Protocol, No Version Negotiation**  
   If `kActionFlagsSerializedLength` changes from 4 to 8 bytes, all clients must update. There's no version field in packets to signal the change. A client on an old build will misparse hub packets.

2. **CRC-16 Is Not Cryptographic**  
   The 2-byte CRC header (`kStarPacketHeaderSize = 4`: 2 magic + 2 CRC) prevents accidental corruption but not intentional forgery. If the hub is exposed to untrusted networks (e.g., internet metaserver), an attacker can forge packets by brute-forcing the CRC space (~65k combinations).

3. **No Explicit Latency Compensation**  
   `spoke_latency()` estimates RTT, but it's not clear how the hub orders actions from clients with widely different latencies. If player A has 50ms latency and player B has 500ms, do simultaneous inputs from both arrive in different ticks at the hub? This could cause non-determinism or unfair input precedence.

4. **Pregame Synchronization is Hardcoded**  
   `kPregameTicks = TICKS_PER_SECOND * 3 = 90 ticks`. If a client has >300ms latency (3 seconds), initial synchronization may timeout or fail silently. No adaptive retry or backoff.

5. **Circular Queue Overflow Not Addressed**  
   `WritableTickBasedActionQueue` is circular, but the maximum size isn't declared in this header. If network stalls for too long, old ticks may be silently overwritten. Query via `spoke_get_smallest_unconfirmed_tick()` mitigates this, but there's no assertion that the game consumes fast enough.

6. **No Bandwidth or Jitter Metrics Exposed**  
   `spoke_latency()` returns point-in-time RTT, not histogram or percentile. If latency spikes, the game has no mechanism to slow down, buffer ahead, or notify the player. Compare to modern engines (Unreal, Godot) which track jitter and packet loss.
