# Source_Files/Network/network_star_hub.cpp

## File Purpose
Implements the hub (server) side of a star-topology network protocol for synchronized multiplayer games. Receives action flags from players (spokes), maintains tick-based circular queues, calculates per-player timing offsets, synthesizes flags for lagging players (bandwidth reduction), and broadcasts synchronized game state to all connected players.

## Core Responsibilities
- Initialize/cleanup hub background tasks and state; manage graceful shutdown
- Deserialize incoming game data packets, acknowledgments, identification, and ping requests
- Track per-player connectivity, acknowledgments, ack history, address mapping (NAT-friendly)
- Maintain synchronized tick-based action flag queues for all players
- Calculate and adjust per-player timing offsets using windowed Nth-element statistics
- Handle player net-death detection and propagation
- Synthesize action flags for lagging players (bandwidth reduction feature)
- Serialize and broadcast game state packets with acknowledgments, timing messages, and action flags
- Calculate and report per-player latency and jitter statistics

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `HubPreferences` | struct | Timing windows, net-death thresholds, send periods, Nth-element indices |
| `NetworkPlayer_hub` | struct | Address, connectivity, ACK state, timing adjustment state, latency buffers, network stats |
| `AddressToPlayerIndexType` | typedef | `std::map<IPaddress, int>` ΓÇö maps IP address to player index for NAT-friendly identification |
| `TickBasedActionQueueCollection` | typedef | `std::vector<TickBasedActionQueue>` ΓÇö one action flag queue per player |
| `MutableElementsTickBasedCircularQueue<uint32>` | template class | Allows mutation of enqueued elements; used for bitmask tracking of pending ACKs |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sNetworkTicker` | int32 | static | Network clock, advances every tick independent of game clock |
| `sSmallestRealGameTick` | int32 | static | First tick of actual game (pregame ticks = 3 sec before this) |
| `sFlagsQueues` | TickBasedActionQueueCollection | static | Action flags for each player, indexed by tick |
| `sLateFlagsQueues` | TickBasedActionQueueCollection | static | Late-arriving flags from lagging players, kept separate for collapse/merge |
| `sPlayerDataDisposition` | MutableElementsTickBasedCircularQueue<uint32> | static | Bitmask per tick: which players still owe data or ACKs |
| `sPlayerReflectedFlags` | MutableElementsTickBasedCircularQueue<uint32> | static | Bitmask per tick: which players had flags synthesized (reflected back to them) |
| `sSmallestIncompleteTick` | int32 | static | Oldest tick awaiting player acknowledgments |
| `sSmallestUnsentTick` | int32 | static | Oldest tick not yet sent (used for bandwidth reduction batching) |
| `sConnectedPlayersBitmask` | uint32 | static | Bitmask of all currently connected players |
| `sLaggingPlayersBitmask` | uint32 | static | Bitmask of players with outstanding incomplete ticks (triggers flag synthesis) |
| `sNetworkPlayers` | NetworkPlayerCollection | static | Array of per-player state |
| `sAddressToPlayerIndex` | AddressToPlayerIndexType | static | Address-to-index mapping discovered from incoming packets |
| `sFlagSendTimeQueue` | ConcreteTickBasedCircularQueue<int32> | static | Records `sNetworkTicker` when each tick's flags first sent (latency measurement) |
| `sOutgoingFrame` | UDPpacket | static | Reused buffer for outgoing packets |
| `sHubActive` | std::atomic_bool | static | Flag: hub is receiving/processing packets |
| `sHubTickTask` | myTMTaskPtr | static | Handle to background tick task |

## Key Functions / Methods

### hub_initialize
- **Signature**: `void hub_initialize(int32 inStartingTick, int inNumPlayers, const IPaddress* const* inPlayerAddresses, int inLocalPlayerIndex)`
- **Purpose**: Initialize hub state, allocate queues, set timing preferences, and spawn background tick task
- **Inputs**: Starting game tick, number of players, addresses (may be NULL for unknown), local player index (NONE for standalone)
- **Outputs/Return**: None
- **Side effects**: Allocates circular queues, resets all player state, sets `sHubActive = true`, spawns `myXTMSetup()` task at ~30 Hz, opens debug log if enabled
- **Calls**: `myXTMSetup()`
- **Notes**: Thread-safe via `mytm_mutex` for subsequent packet processing; pregame = 3 seconds of warmup ticks before actual game starts

### hub_cleanup
- **Signature**: `void hub_cleanup(bool inGraceful, int32 inSmallestPostGameTick)`
- **Purpose**: Gracefully or forcefully shut down hub and block until tasks finish
- **Inputs**: Graceful flag (true = wait for all players to ACK game-end), game-end tick
- **Outputs/Return**: None
- **Side effects**: Sets `sHubActive = false`, calls `myTMRemove()`, `myTMCleanup()`, clears all queues and player state, closes debug log
- **Calls**: `take_mytm_mutex()`, `hub_check_for_completion()`, `release_mytm_mutex()`, `myTMRemove()`, `myTMCleanup()`
- **Notes**: Graceful path waits in loop for `!sHubActive` before fully cleaning up

### hub_received_network_packet
- **Signature**: `void hub_received_network_packet(UDPpacket& inPacket, bool from_local_spoke = false)`
- **Purpose**: Entry point for all incoming packets; validates CRC, routes to handlers
- **Inputs**: UDP packet buffer, flag indicating local player source
- **Outputs/Return**: None
- **Side effects**: Deserializes magic/CRC, validates packet, increments error stats on CRC failure, dispatches to game data / identification / ping handlers
- **Calls**: Packet handler functions, `NetDDPSendFrame()`, `spoke_received_network_packet()`, `check_send_packet_to_spoke()`
- **Notes**: Ignores non-ping packets if `!sHubActive`; catches and silently discards malformed packets

### hub_received_game_data_packet_v1
- **Signature**: `static void hub_received_game_data_packet_v1(AIStream& ps, int inSenderIndex)`
- **Purpose**: Deserialize and process game data from a player: ACK, messages, timing data, action flags
- **Inputs**: Deserialization stream, sender index
- **Outputs/Return**: None
- **Side effects**: Calls `player_acknowledged_up_to_tick()`, processes messages, updates timing window and calculates arrival offsets, enqueues action flags and late flags, may call `send_packets()`
- **Calls**: `player_acknowledged_up_to_tick()`, `process_messages()`, `player_provided_flags_from_tick_to_tick()`, `send_packets()`
- **Notes**: Handles pregameΓåÆingame window size transition; late flags kept in separate queue, merged when player catches up; logs timing adjustment decisions for debugging

### hub_tick
- **Signature**: `static bool hub_tick()`
- **Purpose**: Periodic background task (~30 Hz) to detect net-dead players, synthesize flags for lagging players, send recovery packets, update latency/jitter stats
- **Inputs**: None
- **Outputs/Return**: Always `true` (continue running)
- **Side effects**: Increments `sNetworkTicker`, calls `make_player_netdead()` for silent players, calls `make_up_flags_for_first_incomplete_tick()` if bandwidth reduction enabled, sends packets, updates per-player latency/jitter
- **Calls**: `make_player_netdead()`, `make_up_flags_for_first_incomplete_tick()`, `send_packets()`, `check_send_packet_to_spoke()`
- **Notes**: Registered as callback to `myXTMSetup()` for periodic invocation

### send_packets
- **Signature**: `static void send_packets()`
- **Purpose**: Serialize and transmit game state packets to all connected players with known addresses
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Records tick in `sFlagSendTimeQueue`, for each player encodes ACK, timing adjustment/net-dead messages, action flags in tick-major order, optionally reflected flags, calls `NetDDPSendFrame()` or `send_frame_to_local_spoke()`, updates `sLastNetworkTickSent` and `sSmallestUnsentTick`
- **Calls**: `NetDDPSendFrame()`, `send_frame_to_local_spoke()`, CRC calculation
- **Notes**: Bandwidth reduction mode can choose incremental (last 3 ticks) or recovery (up to 4 seconds) updates based on effective latency; tick-major encoding allows spoke to decode all players' flags in one pass

### player_acknowledged_up_to_tick
- **Signature**: `static void player_acknowledged_up_to_tick(size_t inPlayerIndex, int32 inSmallestUnacknowledgedTick)`
- **Purpose**: Process player ACK; dequeue completed ticks, update latency stats, check for game completion
- **Inputs**: Player index, smallest unACKed tick they claim
- **Outputs/Return**: None
- **Side effects**: Masks off player bit in `sPlayerDataDisposition` per ACKed tick, when all bits clear dequeues from queues, updates latency buffer and rolling sum, clears outstanding timing adjustment when appropriate, calls `hub_check_for_completion()`
- **Calls**: `hub_check_for_completion()`
- **Notes**: Non-local players contribute latency samples; late ACKs ignored; latency buffer capped at 5 seconds, display window at 1 second

### player_provided_flags_from_tick_to_tick
- **Signature**: `static bool player_provided_flags_from_tick_to_tick(size_t inPlayerIndex, int32 inFirstNewTick, int32 inSmallestUnreceivedTick)`
- **Purpose**: Mark ticks for which player provided data, advance completion boundary, return whether send is warranted
- **Inputs**: Player index, range of newly-received ticks
- **Outputs/Return**: `true` if at least one tick became complete
- **Side effects**: Enqueues missing ticks in `sPlayerDataDisposition` with all-connected mask, masks off player bit for provided ticks, removes player from `sLaggingPlayersBitmask`, increments `sSmallestIncompleteTick` when all players provided data
- **Calls**: None
- **Notes**: Returns `true` when new complete ticks exist, guiding send decision

### make_up_flags_for_first_incomplete_tick
- **Signature**: `static bool make_up_flags_for_first_incomplete_tick()`
- **Purpose**: Synthesize action flags for lagging players to allow game to progress (bandwidth reduction)
- **Inputs**: None
- **Outputs/Return**: `true` if flags synthesized
- **Side effects**: For each lagging player at `sSmallestIncompleteTick`, collapses late flags to midpoint (preserving trigger bits) or synthesizes minimal motion flags, sets bits in `sPlayerReflectedFlags`, advances `sSmallestIncompleteTick`
- **Calls**: `local_random()`
- **Notes**: Reflects flags back to sender so they see their own synthesized input; preserves weapon/action triggers but discards motion bandwidth

### make_player_netdead
- **Signature**: `static void make_player_netdead(int inPlayerIndex)`
- **Purpose**: Declare player disconnected and propagate to other players
- **Inputs**: Player index
- **Outputs/Return**: None
- **Side effects**: Sets `mConnected = false`, clears address mapping, updates `sConnectedPlayersBitmask`, calls `player_provided_flags_from_tick_to_tick()` and `player_acknowledged_up_to_tick()` to fast-track acknowledgments
- **Calls**: `player_provided_flags_from_tick_to_tick()`, `player_acknowledged_up_to_tick()`
- **Notes**: Sets `mNetDeadTick` so outgoing packets include net-dead message; in standalone mode, may cascade-disconnect remaining players

## Control Flow Notes
**Init/Startup**: `hub_initialize()` allocates queues and spawns `sHubTickTask` at ~30 Hz via `myXTMSetup()`.

**Frame/Tick Loop**: 
- `hub_tick()` runs periodically from background task, checking for net-dead players and triggering flag synthesis/sends
- `hub_received_network_packet()` runs in packet handler thread (separate from tick task), deserializing incoming packets and updating flag/ACK queues
- Both access shared state via `mytm_mutex` (mutually exclusive)

**Shutdown**: `hub_cleanup()` sets `sHubActive = false`, removes tick task, waits for completion, and clears state. Graceful mode waits for all player ACKs first.

## External Dependencies
- **Includes**: `network_star.h` (public interface), `TickBasedCircularQueue.h`, `AStream.h` (big-endian I/O), `Logging.h`, `WindowedNthElementFinder.h` (Nth-element statistics), `mytm.h` (task/mutex), `crc.h`, `player.h` (action flag masks)
- **External symbols**: `NetDDPSendFrame()`, `NetGetPlayerData()`, `spoke_received_network_packet()`, `myXTMSetup()`, `myTMRemove()`, `myTMCleanup()`, `take_mytm_mutex()`, `release_mytm_mutex()`, `calculate_data_crc_ccitt()`, `local_random()`
