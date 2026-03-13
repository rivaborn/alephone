# Source_Files/Network/network_star_spoke.cpp - Enhanced Analysis

## Architectural Role

This file implements the client-side ("spoke") component of a **star-topology multiplayer synchronization protocol**, translating between low-level UDP packet I/O and high-level action flag queues consumed by the game simulation loop. It's the critical glue between **GameWorld** (which runs a deterministic 30 FPS simulation) and **Network** (which delivers variable-latency packets). The spoke maintains strict causality: action flags arrive as acknowledged batches, enabling the hub to ensure all players process the same tick range identically. It also implements **NAT-friendly peer identification** (spoke sends self-ID to hub, not the reverse) and **dynamic timing adjustment** via a windowed percentile latency measurement, allowing the game to self-synchronize without explicit time server.

## Key Cross-References

### Incoming (who depends on this file)

| Caller | Usage |
|--------|-------|
| **GameWorld/marathon2.cpp** (`update_world`) | Calls `spoke_tick()` via timer task; queries `spoke_get_net_time()` to adjust game simulation tick; calls `spoke_initialize/cleanup()` for session lifecycle |
| **Input subsystem** (vbl.h, `parse_keymap`) | Called by `spoke_tick()` to consume raw keyboard/input state and generate local player action flags |
| **Rendering/Screen** (HUD, display latency) | Reads `sDisplayLatencyBuffer` to show ping/latency feedback to player |
| **Player subsystem** (player.h, `make_player_really_net_dead()`) | Called when spoke detects net-dead timeout to notify game world |
| **Network packet dispatcher** | `spoke_received_network_packet()` is entry point from network I/O thread for all hubΓåÆspoke packets |

### Outgoing (what this file depends on)

| Dependency | Purpose |
|------------|---------|
| **network_star.h** | Protocol constants (packet magic, message types), hub interface (`hub_received_network_packet()`) |
| **AStream.h** (AIStreamBE, AOStreamBE) | Big-endian binary serialization for cross-platform packet parsing/generation |
| **mytm.h** (timer task API) | Periodic callback scheduling (`myXTMSetup()`, `myTMRemove()`) for spoke_tick at ~30 FPS |
| **crc.h** (`calculate_data_crc_ccitt`) | CRC-16 validation to detect corrupted packets before processing |
| **vbl.h** (`parse_keymap`) | Converts raw input state (keyboard, mouse, gamepad) to action flags |
| **player.h** (`make_player_really_net_dead()`) | Notify game world when remote player times out |
| **Logging.h** | Diagnostic logging with contexts (NMT = Network Message Trace) |
| **InfoTree.h** | Load tunable preferences (timing window size, net-death thresholds) |
| **WindowedNthElementFinder** | Maintains rolling window of latency samples; extracts nth-largest for robust timing adjustment |

## Design Patterns & Rationale

### 1. **Dual-Queue Action Flag Buffer** (`sOutgoingFlags` + `sUnconfirmedFlags` + `sLocallyGeneratedFlags`)

Three logically separate queues manage flag lifecycle:
- `sOutgoingFlags`: Flags already sent to hub, awaiting ACK (kept for retransmit if packets lost)
- `sUnconfirmedFlags`: Flags generated locally but not yet sent (only fed to hub when needed)
- `sLocallyGeneratedFlags`: Virtual composite queue (children = both above) for iteration during local tick

**Rationale**: Decouples **generation** (local input ΓåÆ unconfirmed queue) from **transmission** (unconfirmed ΓåÆ outgoing via network packet). Enables "recovery send period" (resend older flags without regenerating), crucial for lossy networks. Hub's piggybacked ACK allows dequeueing confirmed flags without blocking on remote round-trip.

### 2. **Windowed Nth-Element Latency Measurement**

Instead of averaging latency (vulnerable to outliers), spoke maintains a **3-second rolling window** of `(write_tick - unread_tick)` samples, extracts the **median/percentile**, and applies that as timing adjustment. Computed once per full window via `WindowedNthElementFinder<int32>`.

**Rationale**: Median-based measurement is robust to transient spikes (e.g., single dropped packet causing temporary queue overflow). Hub can then request spoke to "speed up" (positive adjustment = skip ticks) or "slow down" (negative = provide extra) to keep latencies balanced across the game session.

### 3. **NAT-Friendly Peer Identification** (Sept 2004 evolution)

Spokes send **identification packets** containing their player ID; hub learns their network address from the **packet source IP/port**, not from a topology announcement upfront.

**Rationale**: Pre-2004 design assumed hub could reach spokes directly via announced addresses; broke behind NAT/firewalls. New design: only hubΓåÆspoke traffic requires reachability; spoke can be behind any firewall and still send (UDP is one-way). Hub never initiates contact; all packets are responses to spoke identification/data.

### 4. **"We Are Alone" Special Case**

When the only connected player is this spoke (all others are zombies or have net-dead ticks in the past), enqueue `NET_DEAD_ACTION_FLAG` for other players up to the most recent ACK'd tick. This prevents the game from stalling when waiting for non-existent peers.

**Rationale**: Subtle state machine. Without this, if only one player is alive, the game loop would hang on `sOutgoingFlags.getReadTick()` waiting for flags for dead players that will never arrive. The check `(sNetworkPlayers[i].mConnected || sNetworkPlayers[i].mNetDeadTick > theSmallestUnacknowledgedTick)` avoids enqueueing if a player is permanently dead (netDeadTick in past).

### 5. **Pregame vs. In-Game Timeout Thresholds**

- **Pregame**: 90 ticks (~3 seconds at 30 FPS) before net death
- **In-game**: 5 ticks (~166 ms)

**Rationale**: Pregame is forgiving; players may take time to join. In-game requires immediate responsiveness; even brief network lag risks fatal desynchronization.

### 6. **Message Dispatch Table** (`sMessageTypeToMessageHandler`)

Handler function pointers keyed by message type ID; unknown types are silently ignored.

**Rationale**: Decouples message parsing (bitstream reading) from handling. Allows future protocol extensions without recompiling core packet logic. Unknown message types won't crash, just skip.

## Data Flow Through This File

```
ΓöîΓöÇ Network I/O Thread ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé UDP packet arrives        Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö¼ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
           Γöé
           Γû╝ spoke_received_network_packet()
       ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
       Γöé 1. Validate magic + CRC    Γöé (drop if corrupted)
       Γöé 2. Dispatch by packet type Γöé
       ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö¼ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
                    Γöé
         ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö┤ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
         Γöé                          Γöé
         Γû╝ (Game Data)              Γû╝ (Ping)
   spoke_received_                 Ping
   game_data_packet_v1()          handlers
         Γöé
    ΓöîΓöÇΓöÇΓöÇΓöÇΓö╝ΓöÇΓöÇΓöÇΓöÇΓöÉ
    Γöé    Γöé    Γöé
    Γû╝    Γû╝    Γû╝
   ACK  MSG  FLAGS
    Γöé    Γöé    Γöé
    Γöé    Γû╝    Γöé
    Γöé process_Γöé
    Γöé messagesΓöé
    Γöé  Γöé Γû╝    Γöé
    Γöé ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
    Γöé Γöé Timing adjustment?  Γöé
    Γöé Γöé Net-dead notify?    Γöé
    Γöé ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö¼ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
    Γöé          Γöé
    Γû╝          Γû╝
  Dequeue   Apply to
  outgoing  game state
  flags   (if needed)
    Γöé
    ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö¼ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
                    Γû╝
    ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
    Γöé For each player (unread tick): Γöé
    Γöé - Enqueue flags OR             Γöé
    Γöé - Enqueue NET_DEAD_FLAG        Γöé
    ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
                    Γöé
                    Γû╝
         [Player queues updated]
              Γåô
    ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
    Γöé GameWorld/marathon2.cpp  Γöé
    Γöé reads queues at          Γöé
    Γöé "network tick"           Γöé
    ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ

Parallel (Timer-Based):

ΓöîΓöÇ spoke_tick() (called ~30/sec) ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé 1. Increment sNetworkTicker              Γöé
Γöé 2. Check hub silence                    Γöé
Γöé    if overdue ΓåÆ spoke_became_disconnected
Γöé 3. Apply requested timing adjustment    Γöé
Γöé 4. parse_keymap() ΓåÆ generate local flag Γöé
Γöé 5. Enqueue to sUnconfirmedFlags         Γöé
Γöé 6. Check send conditions:               Γöé
Γöé    - New unconfirmed flags exist? send  Γöé
Γöé    - Recovery period elapsed? resend    Γöé
Γöé    - No ACK heard & time for ID? send ID
Γöé 7. send_packet() or send_identification_packet()
Γöé 8. Set sWorldUpdate = true              Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
```

## Learning Notes

1. **Deterministic simulation with variable network latency**: Unlike RTS games that "pause" on packet loss, Aleph One maintains a **deterministic tick-based simulation** where all players process the same tick range. The spoke's job is to **keep queues filled** and timing synchronized; latency is absorbed into the `spoke_get_net_time()` offset, making the game effectively "late" by network RTT.

2. **Retrograde compatibility and NAT evolution**: Code comments (Sept 2004) mark the shift from "topology announces addresses" to "we identify ourselves." This reflects real-world NAT/firewall pain; the codebase is pragmatic about network reality.

3. **Percentile-based latency vs. ping**: Most games report "ping" (RTT); this engine uses **latency in action-ticks**, measured as queue depth. Robust: a brief spike doesn't cause overcorrection. The windowed nth-element is a smart way to detect sustained congestion vs. transient jitter.

4. **Zombies vs. net-dead ticks**: A player is a "zombie" (no local queue exists) if they never connected. A player becomes "net-dead" (queue exists, flags are NET_DEAD_FLAG) after a timeout. The distinction matters for the "we are alone" logic.

5. **Idiomatic to Aleph One era** (early-2000s multiplayer design):
   - Star topology (not peer-to-peer): simpler for latency-sensitive games, easier to prevent cheating.
   - UDP (lossy) + ACK-based retransmit: assumes hub is trusted; no explicit ARQ per packet.
   - Fixed 30 FPS game tick: deterministic physics; timing adjustments are coarse (whole ticks).
   - Big-endian binary format: cross-platform (x86/PPC Macs).

## Potential Issues

1. **Race condition in `sSpokeActive` check** (`spoke_received_network_packet` line ~330):
   - `sSpokeActive` is checked without mutex; during `spoke_cleanup()`, a packet could arrive between `sSpokeActive = false` and mutex release, causing use-after-free if it reads now-freed player queues.
   - **Mitigation in code**: `spoke_cleanup()` holds mutex while setting `sSpokeActive`, but window exists between check and use in packet handler.

2. **Unbounded growth of `sUnconfirmedFlags` queue**:
   - If hub never ACKs (network partition), `sUnconfirmedFlags` grows without bound (no size check before enqueue in `spoke_tick`).
   - No explicit "queue overflow = disconnect" logic visible in this file.

3. **Display latency buffer assumptions**:
   - `sDisplayLatencyBuffer.resize(TICKS_PER_SECOND, 0)` assumes a fixed ring buffer of 1-second samples.
   - Insert logic not visible; unclear if circular wrapping is implemented. Risk of out-of-bounds access if not careful.

4. **Timing window size not tunable at runtime**:
   - `sNthElementFinder` initialized with `kDefaultTimingWindowSize = 3 * TICKS_PER_SECOND`.
   - If loaded from `sSpokePreferences` (XML), window size change mid-game could cause buffer reallocation.

5. **CRC blanking mutates input packet**:
   - `inPacket.buffer[2/3] = 0` before CRC check modifies the caller's buffer; if caller expects immutability, this is a surprise side effect.

6. **No explicit max retry count for identification packets**:
   - `send_identification_packet()` is called repeatedly but no visible limit on how many times before giving up.
   - Could spam network if hub is permanently unreachable.
