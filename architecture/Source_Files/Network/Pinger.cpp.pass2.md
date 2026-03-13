# Source_Files/Network/Pinger.cpp - Enhanced Analysis

## Architectural Role

Pinger implements low-level network latency probing for multiplayer game connectivity assessment. It runs asynchronously to the game loop: ping packets are sent before frame updates, responses arrive via the network receive thread, and round-trip times are collected on-demand with configurable timeout. This isolates latency measurement from the deterministic 30 FPS game simulation (see `GameWorld/marathon2.cpp:update_world`), allowing network quality monitoring without affecting game state consistency.

## Key Cross-References

### Incoming (who depends on this file)
- **Network subsystem initialization** (`network_star.h`, `network_private.h`) ΓÇö likely called from `Network/network_games.cpp` or `Network/Metaserver/network_metaserver.cpp` to measure peer connectivity
- **Game startup/matchmaking** ΓÇö registration and ping calls must occur before or parallel to multiplayer session setup
- **Implicit via network receive thread** ΓÇö `StoreResponse()` called when UDP response packet arrives (callback from `NetDDPSendFrame` / receive handler)

### Outgoing (what this file depends on)
- **`mytm.h` (CSeries timing layer)**:
  - `machine_tick_count()` ΓÇö frame-based tick counter (not wall-clock), critical for deterministic replay
  - `take_mytm_mutex()` / `release_mytm_mutex()` ΓÇö protects UDP send operations from race with receive thread
  - `sleep_for_machine_ticks(1)` ΓÇö cooperative sleep during polling loop
- **`network_star.h` (DDP/UDP abstraction)**:
  - `NetDDPSendFrame()` ΓÇö unreliable UDP transmission
  - `kPingRequestPacket`, `kStarPacketHeaderSize`, `ddpMaxData` ΓÇö packet format constants
- **`crc.h`** ΓÇö `calculate_data_crc_ccitt()` validates ping packets (prevents spoofed responses in `StoreResponse`)
- **`AStream.h`** ΓÇö `AOStreamBE` big-endian serialization for network-safe binary encoding

## Design Patterns & Rationale

**Fire-and-forget with atomic polling**: Ping requests and responses are decoupled across threads ΓÇö main thread sends (mutex-protected), network receive thread stores results (atomic write), query thread polls with timeout. This avoids blocking the game loop on network I/O.

**Monotonic identifier counter**: Uniquely correlates requests to responses despite packet loss/reordering. Pre-increment ensures no ID reuse across `Register()` calls.

**Defensive exception handling**: Try-catch swallows all exceptions during `Ping()`, gracefully degrading if packet serialization or transmission fails ΓÇö does not crash the game.

**Asymmetric atomicity**: `ping_sent_tick` (uint64_t) is NOT atomic; written once per address per `Ping()` call, read only in `GetResponseTime()`. `pong_received_tick` (atomic) guards against race when network thread writes while main thread polls ΓÇö reflects producer/consumer roles.

**Tick-based timing**: Uses `machine_tick_count()` (game frame ticks, typically ~33ms each) rather than wall-clock milliseconds. Enables frame-locked determinism but creates semantic ambiguity: timeout is named `timeout_ms` but compared to tick deltas.

## Data Flow Through This File

1. **Registration phase**: `Register(ip)` ΓåÆ monotonic ID stamped, entry inserted into `_registered_ipv4s` map
2. **Send phase**: `Ping(retries, skip_if_sent)` ΓåÆ serialize ID + CRC into UDP buffer, attempt transmission (1ΓÇôN tries), record `ping_sent_tick`
3. **Receive phase** (network thread): incoming pong packet ΓåÆ `StoreResponse(id, ip)` called by packet handler ΓåÆ atomically write `pong_received_tick`
4. **Query phase**: `GetResponseTime(timeout_ms)` ΓåÆ busy-wait polling until timeout or all responses received ΓåÆ return map of ID ΓåÆ latency (tick delta, returned as `uint16_t`)
5. **Untracked lifecycle**: No deregistration; addresses remain in `_registered_ipv4s` forever

## Learning Notes

**Idiomatic to Aleph One / Marathon era**:
- **Game-tick timing everywhere**: Network layer is frame-locked to game loop ticks (see CSeries `mytm`), not wall-clock. Modern engines separate network and game timing.
- **Coarse-grained mutex protection**: Single global mutex serializes all DDP sends. Scales poorly but simplifies concurrency for era when multiplayer was 2ΓÇô8 players.
- **Binary serialization with CRC**: Pre-computed CRC protects against bitflips (relevant when Marathon ran on slow modems); modern protocols use crypto checksums.
- **Exception-agnostic error handling**: Swallowing exceptions during send implies expected failure modes (e.g., socket closed during shutdown) are acceptable.
- **Polling over callbacks**: `GetResponseTime()` busy-waits rather than triggering callback on response. Simpler to reason about, but wastes CPU.

## Potential Issues

1. **Timeout unit mismatch**: Parameter named `timeout_ms` (milliseconds) but compared to `machine_tick_count()` differences (frame ticks, ~33ms each). If ticks Γëá milliseconds, actual timeout is off by factor of ~33. Verify mapping in `mytm.h`.

2. **No address deregistration**: `Register()` has no corresponding `Unregister()`. Addresses accumulate in `_registered_ipv4s` across game sessions, leaking memory if engine is long-lived or reuses IP addresses.

3. **Inadequate mutex error handling**: `if (take_mytm_mutex())` guards send but continues silently if mutex acquisition fails. Packet not sent, but `ping_sent_tick` not set ΓÇö `GetResponseTime()` will report `UINT16_MAX` (timeout) without distinguishing send failure.

4. **Response time as `uint16_t`**: Maximum representable latency is 65535 ticks (~2 seconds at 30 Hz). Overflows silently wrap if RTT exceeds this.

5. **No TTL/sequence number evolution**: Single ping per address per `Ping()` call. Does not handle duplicate responses or retransmit logic. If a retransmitted packet arrives after initial response, it is ignored (correct); if initial is lost and retry arrives, timing is measured from first send (overstates latency).
