# Source_Files/Network/StarGameProtocol.h

## File Purpose
Defines the interface for a star-topology game protocol handler in the Aleph One game engine. StarGameProtocol implements NetworkGameProtocol for centralized server-based multiplayer games where one server distributes game state to multiple clients.

## Core Responsibilities
- Implement star-topology network synchronization (one authoritative server, multiple clients)
- Initialize and tear down network sessions
- Handle incoming UDP packets in star topology
- Manage action flag prediction and confirmation
- Parse and provide configuration preferences for star protocol behavior
- Query network timing information

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| StarGameProtocol | class | Implements NetworkGameProtocol; manages star-topology multiplayer sessions |
| InfoTree | forward decl. | Configuration tree structure used for preferences (defined elsewhere) |

## Global / File-Static State
None (module-level functions manage state in implementation file).

## Key Functions / Methods

### Enter
- Signature: `bool Enter(short* inNetStatePtr)`
- Purpose: Initialize the star protocol for a network session
- Inputs: Pointer to network state
- Outputs: Boolean indicating success
- Side effects: Initializes internal protocol state
- Calls: Defined in base class, implementation elsewhere

### Sync
- Signature: `bool Sync(NetTopology* inTopology, int32 inSmallestGameTick, int inLocalPlayerIndex, bool isServer)`
- Purpose: Synchronize game state across the network in star topology
- Inputs: Network topology, smallest game tick, local player index, server flag
- Outputs: Boolean indicating sync success
- Side effects: Synchronizes game state with network peers
- Calls: Implementation elsewhere

### UnSync
- Signature: `bool UnSync(bool inGraceful, int32 inSmallestPostgameTick)`
- Purpose: Desynchronize and clean up network session
- Inputs: Graceful shutdown flag, smallest postgame tick
- Outputs: Boolean indicating success
- Side effects: Tears down network connection
- Calls: Implementation elsewhere

### PacketHandler
- Signature: `void PacketHandler(UDPpacket& inPacket)`
- Purpose: Process incoming UDP packets from network peers
- Inputs: Reference to incoming UDP packet
- Outputs: None
- Side effects: Parses and applies packet data to game state
- Calls: Implementation elsewhere

### ParsePreferencesTree (static)
- Signature: `static void ParsePreferencesTree(InfoTree prefs, std::string version)`
- Purpose: Parse configuration preferences for star protocol behavior
- Inputs: Preferences tree, version string
- Outputs: None
- Side effects: Updates protocol preferences/configuration
- Calls: Implementation elsewhere

### GetUnconfirmedActionFlagsCount
- Signature: `int32 GetUnconfirmedActionFlagsCount()`
- Purpose: Query count of unconfirmed action flags used for client-side prediction
- Outputs: Count of unconfirmed flags
- Calls: Implementation elsewhere

### PeekUnconfirmedActionFlag
- Signature: `uint32 PeekUnconfirmedActionFlag(int32 offset)`
- Purpose: Read unconfirmed action flag at given offset without consuming
- Inputs: Offset into unconfirmed flags
- Outputs: Action flag value
- Calls: Implementation elsewhere

### CheckWorldUpdate
- Signature: `bool CheckWorldUpdate() override`
- Purpose: Override of base class; check if world state requires update
- Outputs: Boolean
- Calls: Implementation elsewhere

## Control Flow Notes
This protocol handler integrates into the game's network frame loop. Typical sequence: `Enter()` to initialize ΓåÆ repeated `Sync()` calls per frame ΓåÆ `PacketHandler()` processes incoming updates ΓåÆ `UnSync()` on shutdown. Action flag prediction occurs between Sync calls to minimize perceived latency.

## External Dependencies
- **Base class**: `NetworkGameProtocol` (defines abstract interface)
- **Network types**: `UDPpacket`, `NetTopology`, `int32` (from `network_private.h` via NetworkGameProtocol.h)
- **Configuration**: `InfoTree` class (forward declared; defined elsewhere)
- **Standard**: `<stdio.h>` (likely for logging)
