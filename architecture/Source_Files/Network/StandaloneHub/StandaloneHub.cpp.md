# Source_Files/Network/StandaloneHub/StandaloneHub.cpp

## File Purpose

Implements the StandaloneHub singleton class, which acts as a networked game server accepting connections from gatherer processes (game hosts). It handles connection negotiation, game setup orchestration, and bidirectional message exchange to coordinate game state and data.

## Core Responsibilities

- Singleton lifecycle management (static Init, Reset, Instance access)
- Accept and validate incoming TCP/UDP connections from gatherers
- Perform protocol version and capability negotiation with connecting clients
- Orchestrate the game setup sequence (topology exchange, player data gathering)
- Manage the joiner waiting period and signal game start
- Receive and buffer game-critical messages (topology, physics, map, Lua scripts)
- Provide interfaces for the engine to query received game data
- Send outgoing messages to the connected gatherer

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `StandaloneHub` | class | Singleton hub manager |
| `CommunicationsChannelFactory` | class | Creates server sockets/channels (defined elsewhere) |
| `CommunicationsChannel` | class | Represents one client connection (defined elsewhere) |
| `TopologyMessage`, `PhysicsMessage`, `MapMessage`, `LuaMessage` | classes | Game state/data messages (defined elsewhere) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_instance` | `std::unique_ptr<StandaloneHub>` | static | Singleton instance |

## Key Functions / Methods

### Init
- **Signature:** `static bool Init(uint16 port)`
- **Purpose:** Factory method to create and initialize the singleton hub instance.
- **Inputs:** `port` ΓÇô TCP/UDP port to bind the server socket to.
- **Outputs/Return:** `true` if initialization succeeds or instance already exists; `false` if port is invalid or `NetEnter` fails.
- **Side effects:** Calls `NetEnter(false)` to initialize the network subsystem; creates the singleton instance and server channel factory.
- **Calls:** `NetEnter`, constructor, `CommunicationsChannelFactory` constructor.
- **Notes:** Returns `true` if called when `_instance` already exists (idempotent for existing instances).

### WaitForGatherer
- **Signature:** `bool WaitForGatherer()`
- **Purpose:** Blocks waiting for an incoming connection from a gatherer, then validates protocol version and capabilities.
- **Inputs:** None (uses member state).
- **Outputs/Return:** `true` if a compatible gatherer connects and negotiation succeeds; `false` otherwise.
- **Side effects:** Accepts a connection via `_server->newIncomingConnection()`; receives and stores `RemoteHubHostConnectMessage` and `CapabilitiesMessage`; sends `RemoteHubHostResponseMessage`; flushes outgoing queue. Sets/clears `_gatherer` based on negotiation result.
- **Calls:** `NetSetDefaultInflater`, `receiveSpecificMessage`, `CheckGathererCapabilities`, `NetSetCapabilities`, `enqueueOutgoingMessage`, `flushOutgoingMessages`.
- **Notes:** Uses 3000ms timeout for the connect message, 5000ms for capabilities. Resets `_gatherer` if compatibility check fails.

### SetupGathererGame
- **Signature:** `bool SetupGathererGame(bool& gathering_done)`
- **Purpose:** Orchestrates the full game setup sequence: sends ready signal, processes the joining client, gathers topology and player data, and waits for additional joiners.
- **Inputs:** Reference to `gathering_done` output flag.
- **Outputs/Return:** `true` if setup completes successfully; `false` and calls `Reset()` on failure.
- **Side effects:** Sends `RemoteHubReadyMessage`; calls `NetProcessNewJoiner`, `NetGather`, `GatherJoiners`; transitions `_gatherer` to `_gatherer_client` weak reference; records timestamp in `_start_check_timeout_ms`.
- **Calls:** `enqueueOutgoingMessage`, `NetProcessNewJoiner`, `NetGather`, `GatherJoiners`, `NetCancelGather`, `Reset`.
- **Notes:** Sets `gathering_done = true` only on complete success. Returns early and resets hub on any error.

### GetGameDataFromGatherer
- **Signature:** `bool GetGameDataFromGatherer()`
- **Purpose:** Main message dispatch loop that receives messages from the gatherer and buffers game-critical data.
- **Inputs:** None (uses `_gatherer` or `_gatherer_client`).
- **Outputs/Return:** `true` if `kEND_GAME_DATA_MESSAGE` received and both topology and map messages were buffered; `false` otherwise.
- **Side effects:** Receives and queues messages; stores `TopologyMessage`, `MapMessage`, `PhysicsMessage` in unique_ptr members; forwards Lua messages via `DeferredScriptSend`; deletes unhandled messages.
- **Calls:** `receiveMessage`, `DeferredScriptSend`, message type checkers/getters.
- **Notes:** Handles both compressed (kZIPPED_*) and uncompressed message variants. Discards duplicate topology messages.

### Reset
- **Signature:** `static bool Reset()`
- **Purpose:** Destroys the current singleton instance and reinitializes a new hub on the same port.
- **Inputs:** None.
- **Outputs/Return:** Result of `Init(port)` on the original port.
- **Side effects:** Caches the port, destroys `_instance`, calls `NetExit()`, then calls `Init()` to recreate.
- **Calls:** `NetExit`, `Init`.
- **Notes:** `NetExit()` must be called after destroying the instance.

### GatherJoiners, CheckGathererCapabilities, SendMessageToGatherer
- **Summary:** `GatherJoiners()` polls for new players during a timeout window; `CheckGathererCapabilities()` validates protocol/feature versions (Star, Gatherable, Lua, ZippedData); `SendMessageToGatherer()` locks the weak `_gatherer_client` reference and sends an outgoing message if the client is still alive.

## Control Flow Notes

**Init phase:**
1. User calls static `Init(port)` ΓåÆ creates `CommunicationsChannelFactory` server socket.

**Connection phase:**
2. `WaitForGatherer()` waits for incoming connection, negotiates protocol/capabilities.

**Setup phase:**
3. `SetupGathererGame()` ΓåÆ sends ready signal, gathers topology/player data, waits for additional joiners (5 minutes timeout).

**Data exchange phase:**
4. `GetGameDataFromGatherer()` polls for messages in a loop, buffering game state.

**Teardown:**
5. `Reset()` optionally reinitializes the hub.

## External Dependencies

- **Includes:** `StandaloneHub.h`, `MessageInflater.h`, `network_messages.h`
- **Network layer (defined elsewhere):** `NetEnter`, `NetExit`, `NetSetDefaultInflater`, `NetProcessNewJoiner`, `NetGather`, `NetCancelGather`, `NetCheckForNewJoiner`, `NetSetCapabilities`, `sleep_for_machine_ticks`, `machine_tick_count`
- **Messaging (defined elsewhere):** `CommunicationsChannelFactory`, `CommunicationsChannel`, `Message` subclasses, `Capabilities`
- **Scripting (defined elsewhere):** `DeferredScriptSend`
