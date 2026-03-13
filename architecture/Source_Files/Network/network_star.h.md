# Source_Files/Network/network_star.h

## File Purpose
Header defining the star topology network protocol for Aleph One multiplayer games. Establishes message types, packet structures, and function interfaces for hub (server) and spoke (client) roles communicating via UDP.

## Core Responsibilities
- Define message type constants and packet magic bytes for star topology protocol
- Declare hub initialization, packet handling, and lifecycle management
- Declare spoke (client) initialization, input synchronization, and state queries
- Provide preference configuration interfaces for both hub and spoke modes
- Establish tick-based action queue infrastructure for action flag buffering

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `action_flags_t` | typedef | Uint32 representing player input flags |
| `TickBasedActionQueue` | typedef | Read-only circular queue keyed by game tick |
| `WritableTickBasedActionQueue` | typedef | Write-only interface to action queue |

## Global / File-Static State
None.

## Key Functions / Methods

### hub_initialize
- Signature: `void hub_initialize(int32 inStartingTick, int inNumPlayers, const IPaddress* const* inPlayerAddresses, int inLocalPlayerIndex)`
- Purpose: Initialize the hub (server) with player list and starting network tick
- Inputs: Starting tick, player count, array of player IP addresses, local player index
- Outputs/Return: None
- Side effects: Sets up hub state, network listeners, action queue infrastructure
- Calls: Not inferable from this file
- Notes: Called once at game start; establishes synchronization point with kPregameTicks delay

### hub_cleanup
- Signature: `void hub_cleanup(bool inGraceful, int32 inSmallestPostGameTick)`
- Purpose: Shut down the hub, optionally gracefully
- Inputs: Graceful flag, smallest post-game tick (for validation)
- Outputs/Return: None
- Side effects: Releases hub resources, closes network sockets
- Calls: Not inferable from this file
- Notes: If graceful, communicates shutdown to all spokes

### hub_received_network_packet
- Signature: `void hub_received_network_packet(UDPpacket& inPacket, bool from_local_spoke = false)`
- Purpose: Handle incoming UDP packet from a spoke
- Inputs: UDP packet, flag indicating if from local player
- Outputs/Return: None
- Side effects: Updates hub state, may broadcast to other spokes
- Calls: Not inferable from this file
- Notes: Processes action flags, timing adjustments; validates packet magic

### spoke_initialize
- Signature: `void spoke_initialize(const IPaddress& inHubAddress, int32 inFirstTick, size_t inNumberOfPlayers, WritableTickBasedActionQueue* const inPlayerQueues[], bool inPlayerConnectedStatus[], size_t inLocalPlayerIndex, bool inHubIsLocal)`
- Purpose: Initialize spoke (client) with hub address and local action queues
- Inputs: Hub IP/port, first tick, total player count, writable action queues per player, player connection status array, local player index, hub locality flag
- Outputs/Return: None
- Side effects: Establishes connection to hub, initializes tick-based synchronization
- Calls: Not inferable from this file
- Notes: `inPlayerQueues` array is filled by hub via network; `inHubIsLocal` optimizes local-hub communication

### spoke_received_network_packet
- Signature: `void spoke_received_network_packet(UDPpacket& inPacket)`
- Purpose: Handle incoming UDP packet from hub
- Inputs: UDP packet
- Outputs/Return: None
- Side effects: Updates spoke state, refills action queues, advances network tick
- Calls: Not inferable from this file
- Notes: Validates packet magic and CRC; checks for end-of-message markers

### spoke_get_net_time
- Signature: `int32 spoke_get_net_time()`
- Purpose: Query current network time (in ticks) on spoke
- Inputs: None
- Outputs/Return: Current tick value
- Side effects: None
- Calls: Not inferable from this file
- Notes: May differ from local time due to latency; used for action queue indexing

### spoke_latency
- Signature: `int32 spoke_latency()`
- Purpose: Query estimated round-trip latency to hub
- Inputs: None
- Outputs/Return: Latency in milliseconds; `kNetLatencyInvalid` if not yet measured
- Side effects: None
- Calls: Not inferable from this file
- Notes: Used for client-side prediction; becomes valid after timing adjustment packets received

### spoke_get_unconfirmed_flags_queue
- Signature: `TickBasedActionQueue* spoke_get_unconfirmed_flags_queue()`
- Purpose: Retrieve action queue for local player's unconfirmed flags
- Inputs: None
- Outputs/Return: Pointer to read-only action queue
- Side effects: None
- Calls: Not inferable from this file
- Notes: Used by game logic to read pending inputs; queues are confirmed when hub echoes them back

### spoke_check_world_update
- Signature: `bool spoke_check_world_update()`
- Purpose: Determine if new network state has arrived since last check
- Inputs: None
- Outputs/Return: True if world state updated
- Side effects: May clear internal "dirty" flag
- Calls: Not inferable from this file
- Notes: Signals to game loop whether rerendering or recomputation is needed

## Control Flow Notes
**Initialization Phase:** Hub/spoke are initialized with tick synchronization (kPregameTicks = 3 seconds @ 30 ticks/sec).

**Network Loop (per frame):**
1. Spokes send action flags to hub via UDP packets (magic: S1)
2. Hub collects flags, validates, broadcasts to all via packets (magic: H1 or F1 with spoke flags)
3. Spokes receive hub packets, fill tick-based action queues, update `spoke_check_world_update()` flag
4. Game loop reads action queues, advances tick, re-queries latency

**Shutdown Phase:** Graceful cleanup informs peers; forced cleanup releases immediately.

## External Dependencies
- `TickBasedCircularQueue.h`: `ConcreteTickBasedCircularQueue<T>`, `WritableTickBasedCircularQueue<T>`
- `ActionQueues.h`: Action flag queue management (included but not directly used in this header)
- `NetworkInterface.h`: `IPaddress`, `UDPpacket`, `UDPsocket`
- `map.h`: `TICKS_PER_SECOND` constant (30 ticks/second)
- `<stdio.h>`: Standard I/O (likely for logging/debugging)
- `InfoTree` class (declared forward; defined elsewhere for preferences)
