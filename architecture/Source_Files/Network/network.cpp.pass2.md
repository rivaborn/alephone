# Source_Files/Network/network.cpp - Enhanced Analysis

## Architectural Role

This file is the **central orchestrator of peer-to-peer multiplayer state**, mediating between the transport layer (TCPMess), game simulation (GameWorld), and UI callbacks. It implements a two-tier state machine: the macro level (`netState`: Uninitialized ΓåÆ Gathering ΓåÆ Waiting ΓåÆ Active ΓåÆ Down) governs overall game phases, while micro-level per-client state (`_connecting` ΓåÆ `_connected` ΓåÆ `_ingame`) manages individual joiner lifecycle. Unlike modern game engines with dedicated netcode layers, network.cpp is deeply intertwined with game logicΓÇöit directly reads/writes `dynamic_world`, manages `topology` (the authoritative player array), and orchestrates map distribution. This tight coupling reflects 1990s Marathon architecture where networking was bolted onto a single-player engine rather than designed in.

## Key Cross-References

### Incoming (who calls this file)

- **Game loop** (`marathon2.cpp`: `update_world`) ΓåÆ calls `NetProcessMessagesInGame()` every tick during gameplay
- **Main shell** (`interface.cpp`, `shell.cpp`) ΓåÆ calls `NetInitialize()`, `NetGather()`, `NetChangeMap()` to orchestrate menu/lobby/level transitions
- **LUA scripts** ΓåÆ can invoke `NetGatherPlayer()` to accept joiners (callback entry point)
- **Player input handlers** ΓåÆ calls `NetSendCommand()` to broadcast action flags (Star protocol dispatch)
- **Chat UI** ΓåÆ calls `NetSendChatMessage()` to relay player messages
- **Preferences system** ΓåÆ reads `network_preferences` to determine protocol (Star vs legacy) and network mode

### Outgoing (what this file depends on)

- **TCPMess subsystem** (`CommunicationsChannel.h`, `MessageDispatcher.h`, `MessageInflater.h`) ΓåÆ abstracts TCP transport; dispatches incoming messages by type
- **Network messages** (`network_messages.h`) ΓåÆ serializable message types (HelloMessage, TopologyMessage, MapMessage, etc.)
- **Game protocol** (`StarGameProtocol.h`) ΓåÆ manages game tick sequencing and action flag accumulation
- **Files subsystem** (`game_wad.h`, `interface.h`) ΓåÆ `process_net_map_data()`, `get_map_for_net_transfer()`, `process_network_physics_model()` integrate received data into game state
- **GameWorld** (`map.h`, `player.h`, `dynamic_world`) ΓåÆ reads current map/player state; posts topology changes back to world
- **Console/Logging** ΓåÆ `screen_printf()`, `logAnomaly()` for debug output
- **Metaserver** (`network_metaserver.h`) ΓåÆ announces game/player counts for server discovery
- **Callbacks** (injected by UI) ΓåÆ `GatherCallbacks::JoinSucceeded()`, `ChatCallbacks::ReceivedMessageFromPlayer()` decouple network layer from UI

## Design Patterns & Rationale

**Dual State Machines (Macro + Micro)**  
The macro-level `netState` tracks overall game phases (are we gathering players or playing?), while each connected Client has its own micro state (_connecting, _connected, _awaiting_map, _ingame). This separation avoids a combinatorial explosion of statesΓÇöthe server can be in `netWaiting` (game about to start) while some clients are still in `_awaiting_map` (downloading level data).

**Message Handler Registry via MessageDispatcher**  
Instead of a giant switch statement, each message type has a registered handler object (Client::handleJoinerInfoMessage, etc.). This pattern enabled the 2003 refactoring (split RingGameProtocol.cpp) to support multiple game protocols without touching core message routing.

**Callback Injection for Inversion of Control**  
`GatherCallbacks` and `ChatCallbacks` are injected by the UI layer, not called directly. This prevents circular dependencies between network.cpp and the UI shellΓÇöthe network layer doesn't need to know about menus or chat windows.

**Accumulation + Deferred Processing**  
Map, physics, and Lua script data arrive in chunks via `MapMessage`, `PhysicsMessage`, `LuaMessage`. Rather than process each chunk immediately (which would trigger mid-stream world updates), they're accumulated in static buffers (`handlerMapBuffer`, `handlerPhysicsBuffer`, `handlerLuaBuffer`) then processed together once `EndGameDataMessage` signals completion. This avoids partially-initialized world state.

**Capability Negotiation Gate**  
The `Client::capabilities_indicate_player_is_gatherable()` check happens *before* adding a player to the topology. This early incompatibility detection saves bandwidthΓÇöno sense sending a 50MB map to a client running an old version. The check is extensible: new engine versions add new capability flags (kGameworld, kStar, kLua, kRugby) that can be versioned independently.

## Data Flow Through This File

**Server (Gatherer) Flow:**
1. `NetInitialize()` ΓåÆ creates topology with self at player_index 0, opens listen socket
2. Main loop: `NetCheckForNewJoiner()` polls accept(), processes all client channels
3. For each new connection:
   - Accept ΓåÆ wrap in `Client` object ΓåÆ add to `connections_to_clients` map (keyed by stream_id)
   - Client::Client() ΓåÆ registers message handlers with MessageDispatcher
   - Transport sends `HelloMessage` ΓåÆ client responds with `JoinerInfoMessage`
   - Server calls `Client::capabilities_indicate_player_is_gatherable()` ΓåÆ gate
   - If gatherable: broadcast `ClientInfoMessage(kAdd)` to other clients (for pre-game chat list)
   - Transport sends `JoinPlayerMessage` ΓåÆ client responds with `AcceptJoinMessage`
   - `handleAcceptJoinMessage()` ΓåÆ adds to topology array, calls `gatherCallbacks->JoinSucceeded()`
4. On level change: `NetChangeMap()` ΓåÆ `NetDistributeGameDataToAllPlayers()`
   - Collects all clients by zip capability
   - Deflates map/physics/lua once
   - Broadcasts to each group
   - Clients transition to _ingame
5. During gameplay: `NetProcessMessagesInGame()` pumps all channels; sends `NetworkStatsMessage` every second

**Joiner Flow:**
1. `NetUpdateJoinState()` ΓåÆ loops connecting to server via `ConnectPool::instance()->connect()`
2. On TCP connect:
   - Sets `MessageInflater` for decompression
   - Sets `MessageHandler` ΓåÆ routes to join-phase handlers (not game protocol yet)
3. Join dispatcher routes messages:
   - `handleHelloMessage()` ΓåÆ responds with JoinerInfoMessage
   - `handleCapabilitiesMessage()` ΓåÆ sends capabilities back
   - `handleJoinPlayerMessage()` ΓåÆ user accepted, waits for topology
   - `handleTopologyMessage()` (tag=tagRESUME_GAME or tagPLAYERS) ΓåÆ game start signal
4. On game start signal: `NetChangeMap()` ΓåÆ `NetReceiveGameData()`
   - Waits for `MapMessage`, `PhysicsMessage`, `LuaMessage` chunks
   - Accumulates in static buffers with 60s timeout
   - On `EndGameDataMessage`: process buffers, transition to gameplay
5. During gameplay: `NetProcessMessagesInGame()` pumps single server connection

**Map/Physics Distribution:**
- Server: Obtains map from disk/memory via `get_map_for_net_transfer()`
- Compresses if client supports zipping (capability check)
- Slices into chunks (bounded by MTU, buffering constraints)
- Sends ordered messages: MapMessage(s) ΓåÆ PhysicsMessage(s) ΓåÆ LuaMessage(s) ΓåÆ EndGameDataMessage
- Client: Accumulates ordered messages, validates sequence, processes on EndGameDataMessage
- Deferred processing prevents world state corruption mid-transfer

## Learning Notes

**Why peer-to-peer + server topology?**  
Aleph One uses a hybrid: one machine (the "server" or "gatherer") acts as authority for topology and state synchronization, but players also sync via ring protocol or star protocol for action flags. This is fundamentally different from modern request-response (client sends inputs, server sends world state) or peer-to-peer unordered messaging. The ring protocol, in particular, is elegant for turn-based deterministic simulationΓÇöall players execute identical tick sequences given identical action flag history.

**Why so much global state?**  
The design avoids passing state through deep call chains. `netState`, `topology`, `connections_to_clients` are file-static so they're accessible to all message handlers without plumbing. This reflects pre-OOP C patterns, but it works for a module with clear boundaries (all networking code in one file). Downside: testing is harder without dependency injection.

**Why capability negotiation before gathering?**  
Early 2000s lesson learned: if a joiner claims kStar but doesn't actually support it, they'll desync 10 minutes into the game. By checking capabilities before adding to topology, the gatherer can reject incompatible clients immediately. The kGatherable flag lets joiners pre-opt-out (e.g., "I'm running an old binary, don't bother").

**Why separate map distribution from gathering?**  
Marathon's original design distributed map data during gathering (players could see live map updates before game started). This caused UX issues: long waits for large maps, no clear "game is starting now" moment. The refactored design (gathering ΓåÆ topology lock ΓåÆ then map distribution) provides clear phase transitions that UI can display ("Waiting for players..." ΓåÆ "Receiving map..." ΓåÆ "Game starting!").

## Potential Issues

**Global State Explosion & Testing Burden**  
The module has ~25 file-static globals (topology, connections_to_clients, netState, handlerMapBuffer, etc.). Unit testing requires either mocking all globals or writing integration tests. A modern refactoring would encapsulate these in a NetworkManager class with dependency injection, but the current design reflects 1990s conventions.

**Dangling Pointer Risk in Client Lifecycle**  
Clients are stored in `connections_to_clients[stream_id]` and referenced by handlers. If `Client::drop()` is called while a message handler is executing (e.g., in the middle of pumping channels), the shared_ptr in the map may outlive the channel reference. The code appears safe (handlers don't hold long-lived pointers to channels), but this is subtle and undocumented.

**Sparse Error Recovery in Data Streams**  
`NetReceiveGameData()` waits up to 60 seconds total with 30-second timeouts per message. If the network stalls mid-map, the joiner gets `alert_user()` but there's no graceful reconnectΓÇöthe game is abandoned. Modern implementations would checkpoint and resume, or use a different transport (HTTP range requests).

**Synchronous Stats Collection Under Load**  
`NetProcessMessagesInGame()` on the server collects and distributes `NetworkStatsMessage` every tick (30 FPS). With 8 players, that's 8 stats records sent per frame. No batching or rate limiting visible; could add latency on slow uplinks. The stats are informational (not used for gameplay), so dropping/batching would be safe.

**Race Condition on Resumed Games**  
The `resuming_saved_game` static is set by the gatherer in `NetInitialize()` and read by joiners in `handleTopologyMessage()`. If multiple topology updates arrive during join phase, the flag state could be inconsistent. This is likely safe in practice (topology is sent once), but the reliance on a single flag rather than per-update metadata is fragile.
