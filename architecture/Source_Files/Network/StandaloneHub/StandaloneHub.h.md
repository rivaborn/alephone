# Source_Files/Network/StandaloneHub/StandaloneHub.h

## File Purpose
Defines `StandaloneHub`, a singleton class that manages network communications and game data coordination for a standalone game hub. Acts as the central point for gathering players, validating capabilities, and distributing game topology/map/physics data before game start.

## Core Responsibilities
- Singleton management of the standalone hub instance
- Accept and validate joiner connections and their capabilities
- Retrieve and cache game data (topology, map, physics messages) from the gatherer
- Manage game lifecycle signals (start/end)
- Communicate with gatherer and joiner clients via a communications channel
- Track timeout and game state flags during setup

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `StandaloneHub` | class | Main singleton hub coordinator for game setup |
| `CommunicationsChannelFactory` | class (external) | Creates communication channels |
| `CommunicationsChannel` | class (external) | Bidirectional message channel to a peer |
| `TopologyMessage` | class (external) | Message containing network topology data |
| `MapMessage` | class (external) | Message containing map data |
| `PhysicsMessage` | class (external) | Message containing physics data |
| `Capabilities` | struct (external) | Client capability flags for validation |
| `Message` | class (external) | Base message type for network communication |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_instance` | `static std::unique_ptr<StandaloneHub>` | static | Singleton hub instance |
| `_gathering_timeout_ms` | `static constexpr int` | static | 5-minute timeout for gathering phase |
| `STANDALONE_HUB_VERSION` | `#define` | global | Version string ("01.02") |

## Key Functions / Methods

### Init
- Signature: `static bool Init(uint16 port)`
- Purpose: Initialize the singleton instance with a listen port
- Inputs: `port` ΓÇô TCP port for the hub server
- Outputs/Return: `bool` ΓÇô success/failure
- Side effects: Creates and stores `_instance`
- Calls: (not visible in header)
- Notes: Must be called before `Instance()` is used

### Instance
- Signature: `static StandaloneHub* Instance()`
- Purpose: Retrieve the singleton hub instance
- Inputs: none
- Outputs/Return: Pointer to the singleton, or `nullptr` if not initialized
- Side effects: none
- Calls: none
- Notes: Safe accessor; returns `_instance.get()`

### Reset
- Signature: `static bool Reset()`
- Purpose: Shut down and clear the singleton instance
- Inputs: none
- Outputs/Return: `bool` ΓÇô success/failure
- Side effects: Destroys `_instance`
- Calls: (not visible)
- Notes: Should be called at shutdown

### GetGameDataFromGatherer
- Signature: `bool GetGameDataFromGatherer()`
- Purpose: Fetch topology, map, and physics messages from the gatherer
- Inputs: none
- Outputs/Return: `bool` ΓÇô success if data retrieved
- Side effects: Populates `_topology_message`, `_map_message`, `_physics_message`
- Calls: (not visible)
- Notes: Blocking call; waits for messages from gatherer

### SetupGathererGame
- Signature: `bool SetupGathererGame(bool& gathering_done)`
- Purpose: Coordinate initial game setup with the gatherer (gather joiners, get capabilities, assign players)
- Inputs: `gathering_done` ΓÇô reference to signal when gathering completes
- Outputs/Return: `bool` ΓÇô success/failure
- Side effects: Modifies `_gathering_done`, triggers `GatherJoiners()`, validates capabilities
- Calls: `GatherJoiners()`, `CheckGathererCapabilities()`
- Notes: Core setup flow; manages timeouts via `_start_check_timeout_ms`

### WaitForGatherer
- Signature: `bool WaitForGatherer()`
- Purpose: Block until the gatherer joins the hub
- Inputs: none
- Outputs/Return: `bool` ΓÇô true when gatherer connects
- Side effects: Sets `_gatherer` or `_gatherer_client` depending on connection mode
- Calls: (not visible)
- Notes: Called early in setup flow

### GetGathererChannel
- Signature: `CommunicationsChannel* GetGathererChannel() const`
- Purpose: Return the active communication channel to gatherer
- Inputs: none
- Outputs/Return: Pointer to `_gatherer` or locked `_gatherer_client`
- Side effects: none
- Calls: Calls `.lock()` on weak_ptr if necessary
- Notes: Handles both direct gatherer and client-as-gatherer modes

### SendMessageToGatherer
- Signature: `void SendMessageToGatherer(const Message& message)`
- Purpose: Dispatch a message to the gatherer via its channel
- Inputs: `message` ΓÇô message to send
- Outputs/Return: none
- Side effects: I/O to the network channel
- Calls: (not visible; likely calls `GetGathererChannel()->send()`)
- Notes: Convenience wrapper

### GetMapData / GetPhysicsData
- Signature: `int GetMapData(uint8** data)` / `int GetPhysicsData(uint8** data)`
- Purpose: Extract raw binary data from cached map/physics messages
- Inputs: Pointer-to-pointer for output buffer
- Outputs/Return: `int` ΓÇô size of data in bytes
- Side effects: Allocates or fills buffer (semantic depends on implementation)
- Calls: (not visible)
- Notes: Accessor methods for game setup

### StartGame / SetGameEnded / HasGameEnded
- Signatures: `void StartGame()`, `void SetGameEnded(bool)`, `bool HasGameEnded() const`
- Purpose: Signal and query game lifecycle events
- Side effects: Modify `_start_game_signal` and `_end_game_signal` flags
- Notes: Simple flag setters/getters; used to synchronize with game loop

### SetSavedGame / GathererJoinedAsClient
- Signatures: `void SetSavedGame(bool)`, `void GathererJoinedAsClient()`
- Purpose: Mark game as saved and track gatherer's client status
- Side effects: Modify `_saved_game` and `_gatherer_joined_as_client` flags
- Notes: State markers for post-game or topology decisions

## Control Flow Notes
**Initialization ΓåÆ Game Setup ΓåÆ In-Game ΓåÆ Shutdown:**
1. `Init(port)` ΓåÆ creates server socket, initializes singleton
2. `WaitForGatherer()` ΓåÆ blocks until gatherer connects
3. `SetupGathererGame()` ΓåÆ gathers jointers, validates caps, receives topology
4. `GetGameDataFromGatherer()` ΓåÆ retrieves map & physics before game start
5. `StartGame()` ΓåÆ signals ready to begin
6. During game: `SetGameEnded()` called by game loop on finish
7. `Reset()` ΓåÆ cleanup and shutdown

## External Dependencies
- `MessageInflater.h` ΓÇô message deserialization/inflation machinery
- `network_messages.h` ΓÇô concrete Message subclasses (TopologyMessage, MapMessage, PhysicsMessage, CapabilitiesMessage, etc.)
- `CommunicationsChannelFactory`, `CommunicationsChannel` ΓÇô TCP/network layer (defined elsewhere)
- `Message`, `Capabilities`, `NetTopology` ΓÇô base types (defined elsewhere, likely in network_private.h or network_capabilities.h)
