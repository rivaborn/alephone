# Source_Files/Network/network_star_spoke.cpp

## File Purpose
Implements the client-side ("spoke") component of a star-topology multiplayer network protocol. Handles packet exchange with a central hub server, manages player action flags, detects network death, and adjusts timing for synchronization.

## Core Responsibilities
- Initialize/teardown spoke state with hub address, player queues, and player roster
- Receive and validate game data packets from hub (CRC checking, packet parsing)
- Manage queues of action flags (outgoing, unconfirmed, locally generated) and track acknowledgments
- Enqueue action flags for all players at each tick, generating net-dead flags when necessary
- Measure and apply latency-based timing adjustments to keep client in sync with hub
- Detect hub silence and declare net-dead on timeout
- Send identification packets to hub (NAT-friendly) and periodic data packets with ACKs
- Track display latency for UI feedback
- Parse configuration/preferences (timing windows, net-death thresholds, send periods)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SpokePreferences` | struct | Tunable parameters: pregame/in-game net-death thresholds, recovery send period, timing window size and nth-element index, timing adjustment flag |
| `IncomingGameDataPacketProcessingContext` | struct | State machine for packet processing: tracks message completion and timing adjustment reception |
| `NetworkPlayer_spoke` | struct | Per-player state: zombie flag, connected status, net-dead tick, action queue pointer |
| `MessageTypeToMessageHandler` | typedef std::map | Message type ΓåÆ handler function dispatch table |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sSpokePreferences` | SpokePreferences | static | Configuration for timing/thresholds |
| `sOutgoingFlags` | TickBasedActionQueue | static | Queued action flags waiting for hub ACK |
| `sUnconfirmedFlags` | TickBasedActionQueue | static | Flags generated locally but not yet ACK'd by hub |
| `sLocallyGeneratedFlags` | DuplicatingTickBasedCircularQueue | static | Composite queue (outgoing + unconfirmed) for new action generation |
| `sNetworkPlayers` | vector<NetworkPlayer_spoke> | static | Per-player state (zombie, connected, queues) |
| `sNetworkTicker` | int32 | static | Incremented at each tick; used for timeout detection |
| `sLastNetworkTickHeard` | int32 | static | sNetworkTicker value when last packet arrived from hub |
| `sLastNetworkTickSent` | int32 | static | sNetworkTicker value when last packet sent to hub |
| `sConnected` | bool | static | True if hub is reachable and acknowledged |
| `sSpokeActive` | bool | static | False after cleanup; used to reject incoming packets |
| `sHubAddress` | IPaddress | static | Destination for packets |
| `sLocalPlayerIndex` | size_t | static | Index of this machine's player in the roster |
| `sSmallestUnreceivedTick` | int32 | static | Lowest tick index for which we need flags from hub |
| `sNthElementFinder` | WindowedNthElementFinder<int32> | static | Rolling window of latency measurements; computes nth-largest for timing adjustment |
| `sTimingMeasurementValid` | bool | static | True if nth-element window is full |
| `sTimingMeasurement` | int32 | static | Current latency in ticks (cached from nth-element finder) |
| `sHeardFromHub` | bool | static | True after first game data packet; gates data packet sends vs. identification sends |
| `sDisplayLatencyBuffer` | vector<int32> | static | Ring buffer of recent latency samples (TICKS_PER_SECOND entries) |
| `sSmallestUnconfirmedTick` | int32 | static | Lowest tick in unconfirmed queue not yet moved to player queue |

## Key Functions / Methods

### spoke_initialize
- **Signature:** `void spoke_initialize(const IPaddress& inHubAddress, int32 inFirstTick, size_t inNumberOfPlayers, WritableTickBasedActionQueue* const inPlayerQueues[], bool inPlayerConnected[], size_t inLocalPlayerIndex, bool inHubIsLocal)`
- **Purpose:** Set up spoke state for a new multiplayer session; establish connection to hub and initialize all player queues.
- **Inputs:** Hub address, first game tick, player count, per-player action queues, connection status, local player index, hub locality flag
- **Outputs/Return:** None; initializes static state
- **Side effects:** Resets all queues; starts periodic tick task via myXTMSetup; registers message handlers; allocates player state array
- **Calls:** `myXTMSetup()`, message handler map insertion
- **Notes:** Pregame tick is `inFirstTick - kPregameTicks`; both outgoing and unconfirmed queues reset to pregame start; network players initialized with zombie status based on null queue pointers

### spoke_cleanup
- **Signature:** `void spoke_cleanup(bool inGraceful)`
- **Purpose:** Shut down the spoke and release resources; optionally send final packet to hub.
- **Inputs:** `inGraceful` (unused in this implementation)
- **Outputs/Return:** None
- **Side effects:** Takes mytm mutex; cancels tick task; sets `sSpokeActive = false`; sends one final packet if connected; releases mutex; cleans up all queues and state
- **Calls:** `take_mytm_mutex()`, `myTMRemove()`, `release_mytm_mutex()`, `myTMCleanup()`, `send_packet()`, `check_send_packet_to_hub()`
- **Notes:** Mutex prevents concurrent packet reception/processing during cleanup; intentionally synchronous to ensure all resources freed before return

### spoke_received_network_packet
- **Signature:** `void spoke_received_network_packet(UDPpacket& inPacket)`
- **Purpose:** Main entry point for received packet processing; validate and dispatch based on packet type.
- **Inputs:** UDP packet (buffer, size, source address)
- **Outputs/Return:** None; side effects on player queues, timing state
- **Side effects:** Updates sSpokeActive check; validates CRC; processes game data, ping requests/responses; logs warnings
- **Calls:** Packet-type specific handlers (spoke_received_game_data_packet_v1, spoke_received_ping_request, spoke_received_ping_response)
- **Notes:** CRC validation blanks out CRC field (bytes 2ΓÇô3) before recalculation; ignores packets if not connected or active (except ping); wrapped in try-catch to skip malformed packets

### spoke_received_game_data_packet_v1
- **Signature:** `static void spoke_received_game_data_packet_v1(AIStream& ps, bool reflected_flags)`
- **Purpose:** Unpack game data packet: ACK, messages, and action flags for all players.
- **Inputs:** Stream (packet data), flag indicating if local player's flags are reflected in response
- **Outputs/Return:** None; heavy side effects
- **Side effects:** Dequeues acknowledged outgoing flags; processes messages (timing adjustment, net-dead); enqueues action flags for all players; updates `sSmallestUnreceivedTick`, latency window, and display latency; may handle "we are alone" case
- **Calls:** `process_messages()`, queue enqueue/dequeue/peek operations, `sNthElementFinder.insert()`, `make_player_really_net_dead()`
- **Notes:** Handles three cases for data availability: (1) no flags in packet (checks if alone, enqueues net-dead flags if so), (2) early flags (skipped), (3) normal flags (enqueued tick-by-tick); local player flags confirmed from unconfirmed queue if not reflected; net-dead players get NET_DEAD_ACTION_FLAG; latency = write_tick ΓêÆ unread_tick; nth-element finder maintains sorted window of latencies

### spoke_tick
- **Signature:** `static bool spoke_tick()`
- **Purpose:** Periodic (per-frame) tick: detect hub silence, generate and enqueue local action flags per timing adjustment, send packets.
- **Inputs:** None
- **Outputs/Return:** Always true (request re-scheduling)
- **Side effects:** Increments sNetworkTicker; may disconnect; adjusts timing; enqueues local flags; sends packets; manages queue state
- **Calls:** `spoke_became_disconnected()`, `parse_keymap()`, `send_packet()`, `send_identification_packet()`, `check_send_packet_to_hub()`
- **Notes:** Timing adjustment Γëñ0 means provide extra flags (late); >0 means skip local ticks (early); net-dead timeout differs pregame (90 ticks) vs in-game (5 ticks); sends on flag generation, recovery period expiry, or identification retries; sets `sWorldUpdate = true` to signal game loop

### send_packet
- **Signature:** `static void send_packet()`
- **Purpose:** Serialize outgoing action flags and ACK into network packet; send to hub.
- **Inputs:** None (reads static state)
- **Outputs/Return:** None; I/O side effects
- **Side effects:** Writes to `sOutgoingFrame`; calculates CRC; sends via `NetDDPSendFrame()` or local hub callback; updates `sLastNetworkTickSent`
- **Calls:** `AOStreamBE` stream operators, `calculate_data_crc_ccitt()`, `NetDDPSendFrame()` or `send_frame_to_local_hub()`
- **Notes:** ACK is `sSmallestUnreceivedTick`; only sends outgoing flags if queue non-empty; wrapped in try-catch; CRC blanking ensures correct calculation

### process_messages
- **Signature:** `static void process_messages(AIStream& ps, IncomingGameDataPacketProcessingContext& context)`
- **Purpose:** Deserialize and dispatch message handlers until end-of-messages marker.
- **Inputs:** Stream positioned after ACK field; context to track state
- **Outputs/Return:** None
- **Side effects:** Reads message type from stream; calls appropriate handler; sets context flags
- **Calls:** Handler functions from `sMessageTypeToMessageHandler` map
- **Notes:** Message dispatch is data-driven; unknown message types ignored (not in map)

### handle_end_of_messages_message
- **Signature:** `static void handle_end_of_messages_message(AIStream& ps, IncomingGameDataPacketProcessingContext& context)`
- **Purpose:** Mark end of message sequence.
- **Inputs:** (Unused stream); context
- **Outputs/Return:** None
- **Side effects:** Sets `context.mMessagesDone = true`
- **Notes:** Terminates message loop in process_messages

### handle_timing_adjustment_message
- **Signature:** `static void handle_timing_adjustment_message(AIStream& ps, IncomingGameDataPacketProcessingContext& context)`
- **Purpose:** Receive timing adjustment from hub; queue it for application in spoke_tick.
- **Inputs:** Stream containing int8 adjustment value; context
- **Outputs/Return:** None
- **Side effects:** Reads adjustment; updates `sOutstandingTimingAdjustment` and `sRequestedTimingAdjustment` if changed
- **Calls:** Logging
- **Notes:** Negative adjustment = provide extra ticks (we're late); positive = skip ticks (we're early)

### spoke_get_net_time
- **Signature:** `int32 spoke_get_net_time()`
- **Purpose:** Return the current network tick for game logic, accounting for latency adjustment if enabled.
- **Inputs:** None
- **Outputs/Return:** Adjusted write tick (if connected and timing valid) or unadjusted local queue write tick
- **Side effects:** Caches previous delay for logging (one-time log on change)
- **Calls:** Logging
- **Notes:** Used by game to query current "network tick" for syncing local state with expected hub state

---

## Control Flow Notes

**Initialization:** `spoke_initialize()` ΓåÆ periodic task spawned via `myXTMSetup()` calling `spoke_tick()` every ~33 ms.

**Per-Tick Flow:**
1. `spoke_tick()` fires (callback from timer task)
2. Apply timing adjustment: generate and enqueue local action flags
3. If new flags or recovery period elapsed, `send_packet()` to hub
4. Set `sWorldUpdate = true` to signal game loop

**Packet Reception:**
1. Network thread delivers `UDPpacket` to `spoke_received_network_packet()`
2. Validate magic, CRC; dispatch by type
3. Game data packets ΓåÆ `spoke_received_game_data_packet_v1()` ΓåÆ ACK processing, message dispatch, flag enqueue ΓåÆ triggers action on player queues
4. Ping packets ΓåÆ `spoke_received_ping_request/response()`

**Disconnect Detection:**
- `spoke_tick()` checks: if `sNetworkTicker ΓêÆ sLastNetworkTickHeard > threshold` ΓåÆ `spoke_became_disconnected()`
- Threshold = 90 ticks pregame, 5 ticks in-game

**Latency Measurement:**
- Each received tick increments `sSmallestUnreceivedTick` ΓåÆ latency = `sOutgoingFlags.getWriteTick() ΓêÆ sSmallestUnreceivedTick`
- Inserted into `sNthElementFinder` (window size = 3 sec default)
- nth-largest element extracted and cached in `sTimingMeasurement` when window full
- Used by `spoke_get_net_time()` to adjust game simulation tick

## External Dependencies
- **AStream.h:** `AIStreamBE`, `AOStreamBE` for binary serialization/deserialization (big-endian)
- **mytm.h:** `myXTMSetup()`, `myTMRemove()`, `myTMCleanup()`, mutex operations for timer tasks
- **network_private.h:** `kPROTOCOL_TYPE`, `NET_DEAD_ACTION_FLAG` constant
- **WindowedNthElementFinder.h:** Windowed percentile calculation for latency statistics
- **vbl.h:** `parse_keymap()` to read local player input (defined elsewhere)
- **CircularByteBuffer.h:** Included but not directly used in this file
- **Logging.h:** Logging macros (`logContextNMT`, `logTraceNMT`, `logDumpNMT`, `logWarningNMT`)
- **crc.h:** `calculate_data_crc_ccitt()` for packet validation
- **player.h:** `make_player_really_net_dead()` (external function)
- **InfoTree.h:** Configuration tree for preferences
- **std::map:** Message type dispatch table
- **Defined elsewhere:** `hub_received_network_packet()`, `NetDDPSendFrame()`, `NetGetPinger()`, `TICKS_PER_SECOND`, `UDPpacket` structure
