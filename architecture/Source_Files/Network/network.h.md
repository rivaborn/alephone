# Source_Files/Network/network.h

## File Purpose
Public network subsystem interface for the Aleph One game engine. Exposes APIs for multiplayer game setup (player gathering, joining), in-game synchronization, chat, and network diagnostics. Implements a state machine for network session lifecycle.

## Core Responsibilities
- Define network session states and state transitions
- Player gathering (host) and joining (client) mechanics
- Chat and event callbacks for UI integration
- Game data distribution and synchronization
- Network diagnostics (latency, jitter, error tracking)
- UDP socket management and packet handling
- Capabilities negotiation between clients
- Remote hub (server relay) support for NAT traversal

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `game_info` | struct | Game configuration (level, mode, difficulty, time/kill limits, random seed) |
| `player_info` | struct | Player identity (name, color, team, serial number) |
| `prospective_joiner_info` | struct | Joining player state (stream ID, name, color, team, gathering flag) |
| `GatherCallbacks` | class (abstract) | Virtual hooks for gather events (player arrived/dropped, join succeeded) |
| `ChatCallbacks` | class (abstract) | Virtual hooks for chat reception and sending |
| `InGameChatCallbacks` | class | Concrete implementation of chat callbacks for in-game UI |
| `RemoteHubCommand` | enum class | Commands sent to remote hub (accept joiner, start/end game) |
| `NetworkStats` | struct | Per-player stats (latency, jitter, error count, pregame state) |

## Global / File-Static State
None.

## Key Functions / Methods

### NetEnter / NetExit
- **Signature:** `bool NetEnter(bool use_remote_hub)` / `void NetExit(void)`
- **Purpose:** Initialize/shutdown network subsystem; optionally enable remote hub for NAT traversal
- **Inputs:** `use_remote_hub` flag
- **Outputs/Return:** `NetEnter` returns success; `NetExit` returns void
- **Side effects:** Sets initial state to `netUninitialized` or enables remote hub mode
- **Calls:** Initializes internal state, may spawn pinger

### NetGather
- **Signature:** `bool NetGather(void *game_data, short game_data_size, void *player_data, short player_data_size, bool resuming_game, bool attempt_upnp)`
- **Purpose:** Host a multiplayer game; listen for joining players
- **Inputs:** Game configuration data, player data template, resume flag, UPnP attempt flag
- **Outputs/Return:** `true` if gathering started successfully
- **Side effects:** Transitions to `netGathering` state; opens listener socket
- **Calls:** Callback invocation for new joiners

### NetGameJoin / NetCheckForNewJoiner / NetProcessNewJoiner
- **Signature:** `bool NetGameJoin(void *player_data, short player_data_size, const char* host_address_string)` / `bool NetCheckForNewJoiner(...)` / `bool NetProcessNewJoiner(...)`
- **Purpose:** Client-side join flow: initiate join, poll for new joiners (gatherer-side), process incoming joiner
- **Inputs:** Player data, host address (join); factory/processing flag (check); communications channel (process)
- **Outputs/Return:** `true` if join initiated or new joiner available
- **Side effects:** Transitions to `netConnecting`/`netJoining` states; creates TCP connections
- **Calls:** Network callbacks, channel management

### NetState
- **Signature:** `short NetState(void)`
- **Purpose:** Query current network session state
- **Outputs/Return:** Current state enum (e.g., `netGathering`, `netActive`, `netDown`)
- **Notes:** Never assigned pseudo-states like `netPlayerAdded`; used to poll session progress

### NetSync / NetStart
- **Signature:** `bool NetSync(void)` / `bool NetStart(void)`
- **Purpose:** Synchronize player topology and readiness; commence game play
- **Outputs/Return:** `true` on success
- **Side effects:** Transitions to `netStartingUp` ΓåÆ `netActive`
- **Calls:** Topology validation, state distribution

### NetDistributeGameDataToAllPlayers / NetReceiveGameData
- **Signature:** `OSErr NetDistributeGameDataToAllPlayers(byte* wad_buffer, int32 wad_length, bool do_physics, CommunicationsChannel* remote_hub = nullptr)` / `byte* NetReceiveGameData(bool do_physics)`
- **Purpose:** Distribute/receive synchronized game state (map, physics, Lua) across network
- **Inputs:** WAD buffer, physics flag, optional remote hub channel
- **Outputs/Return:** Error code / received WAD buffer
- **Side effects:** Broadcasts or receives state; may apply physics updates
- **Calls:** Compression, distribution, message queueing

### NetGetLatency / NetGetStats
- **Signature:** `int32 NetGetLatency()` / `const NetworkStats& NetGetStats(int player_index)`
- **Purpose:** Retrieve network diagnostics (latency in ms, jitter, error count)
- **Inputs:** Player index (stats)
- **Outputs/Return:** Latency or full stats struct
- **Notes:** Returns sentinel values for invalid/disconnected state

### NetSetGatherCallbacks / NetSetChatCallbacks
- **Signature:** `void NetSetGatherCallbacks(GatherCallbacks *gc)` / `void NetSetChatCallbacks(ChatCallbacks *cc)`
- **Purpose:** Register callback handlers for gather/chat events
- **Inputs:** Callback handler pointers
- **Side effects:** Stores pointers for invocation during event dispatch

## Control Flow Notes

**State Machine:**  
Session lifecycle follows: `netUninitialized` ΓåÆ (host) `netGathering` ΓåÆ `netWaiting` ΓåÆ `netStartingUp` ΓåÆ `netActive` ΓåÆ `netDown`  
Or (client) `netUninitialized` ΓåÆ `netConnecting` ΓåÆ `netJoining` ΓåÆ `netWaiting` ΓåÆ `netStartingUp` ΓåÆ `netActive` ΓåÆ `netDown`

**Frame Integration:**  
- `NetUpdateJoinState()` polled during join phase (returns pseudo-states for UI feedback)
- `NetProcessMessagesInGame()` called per-frame in active state to dispatch messages
- `NetGetUnconfirmedActionFlags()` / `NetUpdateUnconfirmedActionFlags()` support client-side prediction

**Remote Hub:**  
`NetConnectRemoteHub()` and `NetRemoteHubSendCommand()` support relay server mode for NAT-traversal scenarios.

## External Dependencies

- **cseries.h, cstypes.h:** Basic types (`uint16`, `int16`, `byte`, `OSErr`, `Rect`)
- **CommunicationsChannel.h:** TCP socket abstraction (`TCPsocket`, `IPaddress`, message queuing)
- **network_capabilities.h:** Capability flags for protocol versioning (gameworld, Lua, zipped data, etc.)
- **Pinger.h:** ICMP latency diagnostics (`NetCreatePinger()`, `NetGetPinger()`)
- Standard library: `<string>`, `<vector>`, `<memory>` (for `std::shared_ptr`, `std::weak_ptr`)
- Symbols defined elsewhere: `entry_point`, `player_start_data`, `UDPpacket`, `NetworkInterface`, `TCPlistener`, `MessageInflater`, `MessageHandler`
