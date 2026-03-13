# Source_Files/Network/StarGameProtocol.cpp

## File Purpose
Glue layer implementing the `StarGameProtocol` class, which interfaces the star-topology network protocol (hub-and-spoke architecture) with the Aleph One game engine. Manages player action queue synchronization, packet routing, and network lifecycle (enter/sync/unsync).

## Core Responsibilities
- Implement network protocol interface (`Enter`, `Sync`, `UnSync`, `PacketHandler`)
- Bridge legacy action queues to tick-based circular queues via adapter template
- Route UDP packets to hub (server) or spoke (client) handlers based on local topology role
- Initialize and clean up hub-side and spoke-side network state
- Track and propagate player network-dead status
- Manage configuration preferences for both hub and spoke

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `LegacyActionQueueToTickBasedQueueAdapter<tValueType>` | Template class | Wraps legacy player action queues to expose tick-based circular queue interface for network protocol |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sStarQueues` | `WritableTickBasedActionQueue*[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS]` | static | Action queue per connected player; NULL for disconnected slots |
| `sTopology` | `NetTopology*` | static | Current active network topology (players, addresses, roles); NULL when not in sync |
| `sNetStatePtr` | `short*` | static | Pointer to external net state variable (e.g., `netActive`, `netDown`); set by caller |
| `sHubIsLocal` | `bool` | static | True if hub runs locally (server), false if remote (client); conditionally compiled or runtime-assigned |

## Key Functions / Methods

### StarGameProtocol::Enter
- **Signature:** `bool Enter(short* inNetStatePtr)`
- **Purpose:** Initialize protocol and store reference to external network state variable.
- **Inputs:** `inNetStatePtr` ΓÇö pointer to external network state variable.
- **Outputs/Return:** `true` (always succeeds).
- **Side effects:** Sets `sNetStatePtr`, optionally initializes `sHubIsLocal` to false if not standalone hub.
- **Calls:** None.
- **Notes:** Must be called before `Sync`.

### StarGameProtocol::Sync
- **Signature:** `bool Sync(NetTopology* inTopology, int32 inSmallestGameTick, int inLocalPlayerIndex, bool isServer)`
- **Purpose:** Set up network protocol with game topology, initial tick, and player role; initialize hub or spoke state.
- **Inputs:** `inTopology` (network topology), `inSmallestGameTick` (starting tick), `inLocalPlayerIndex` (local player, NONE for server), `isServer` (whether this machine is the server).
- **Outputs/Return:** `true` (always succeeds).
- **Side effects:** Creates `LegacyActionQueueToTickBasedQueueAdapter` for each connected player; calls `hub_initialize` or `spoke_initialize`; sets `*sNetStatePtr = netActive`.
- **Calls:** `hub_initialize`, `spoke_initialize`, `GetRealActionQueues()->resetQueue`, `GetRealActionQueues()->availableCapacity`.
- **Notes:** Asserts `inTopology != NULL`; builds adapter queues only for non-NONE player identifiers.

### StarGameProtocol::UnSync
- **Signature:** `bool UnSync(bool inGraceful, int32 inSmallestPostgameTick)`
- **Purpose:** Clean up network protocol, free adapter queues, and stop hub/spoke.
- **Inputs:** `inGraceful` (clean vs. abrupt shutdown), `inSmallestPostgameTick` (final tick for hub postgame).
- **Outputs/Return:** `true` (always succeeds).
- **Side effects:** Calls `spoke_cleanup` and/or `hub_cleanup`; deletes and nullifies `sStarQueues` entries; sets `*sNetStatePtr = netDown`.
- **Calls:** `spoke_cleanup`, `hub_cleanup`.
- **Notes:** Only acts if state is `netStartingUp` or `netActive`.

### StarGameProtocol::PacketHandler
- **Signature:** `void PacketHandler(UDPpacket& packet)`
- **Purpose:** Route incoming UDP packet to hub or spoke handler.
- **Inputs:** `packet` ΓÇö received UDP packet.
- **Outputs/Return:** None.
- **Side effects:** Calls `hub_received_network_packet` or `spoke_received_network_packet`.
- **Calls:** `hub_received_network_packet`, `spoke_received_network_packet`.
- **Notes:** Dispatch based on `sHubIsLocal` flag.

### StarGameProtocol::GetNetTime
- **Signature:** `int32 GetNetTime()`
- **Purpose:** Query current network time from spoke.
- **Inputs:** None.
- **Outputs/Return:** Current network tick (int32).
- **Side effects:** None.
- **Calls:** `spoke_get_net_time`.

### StarGameProtocol::GetUnconfirmedActionFlagsCount
- **Signature:** `int32 GetUnconfirmedActionFlagsCount()`
- **Purpose:** Return count of unconfirmed (unacknowledged) action flags pending.
- **Inputs:** None.
- **Outputs/Return:** Count = `getWriteTick() - getReadTick()`.
- **Side effects:** None.
- **Calls:** `spoke_get_unconfirmed_flags_queue`, queue peek/read operations.

### StarGameProtocol::PeekUnconfirmedActionFlag
- **Signature:** `uint32 PeekUnconfirmedActionFlag(int32 offset)`
- **Purpose:** Inspect unconfirmed action flag at read position + offset without consuming.
- **Inputs:** `offset` ΓÇö relative offset from current read tick.
- **Outputs/Return:** `uint32` action flags at that position.
- **Side effects:** None.
- **Calls:** `spoke_get_unconfirmed_flags_queue`, queue `peek`.

### StarGameProtocol::UpdateUnconfirmedActionFlags
- **Signature:** `void UpdateUnconfirmedActionFlags()`
- **Purpose:** Consume (dequeue) unconfirmed action flags up to the smallest unconfirmed tick acknowledged by hub.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Advances queue read pointer.
- **Calls:** `spoke_get_unconfirmed_flags_queue`, `spoke_get_smallest_unconfirmed_tick`, queue `dequeue`.

### StarGameProtocol::CheckWorldUpdate
- **Signature:** `bool CheckWorldUpdate()`
- **Purpose:** Check whether a new world state update is available from hub.
- **Inputs:** None.
- **Outputs/Return:** `true` if update available.
- **Side effects:** None.
- **Calls:** `spoke_check_world_update`.

### make_player_really_net_dead
- **Signature:** `void make_player_really_net_dead(size_t inPlayerIndex)`
- **Purpose:** Mark a player as network-dead in the current topology.
- **Inputs:** `inPlayerIndex` ΓÇö player array index.
- **Outputs/Return:** None.
- **Side effects:** Sets `sTopology->players[inPlayerIndex].net_dead = true`.
- **Calls:** None.
- **Notes:** Asserts `inPlayerIndex < sTopology->player_count`.

### StarGameProtocol::ParsePreferencesTree
- **Signature:** `void ParsePreferencesTree(InfoTree prefs, std::string version)`
- **Purpose:** Parse XML/INI preferences for both hub and spoke.
- **Inputs:** `prefs` (configuration tree), `version` (config format version).
- **Outputs/Return:** None.
- **Side effects:** Delegates to `HubParsePreferencesTree` and `SpokeParsePreferencesTree`.
- **Calls:** `HubParsePreferencesTree`, `SpokeParsePreferencesTree`.

### DefaultStarPreferences / StarPreferencesTree
- **Signature:** `void DefaultStarPreferences()` / `InfoTree StarPreferencesTree()`
- **Purpose:** Load default configuration or construct preferences tree for hub and spoke.
- **Inputs:** None.
- **Outputs/Return:** `InfoTree` with hub/spoke subtrees.
- **Side effects:** None (StarPreferencesTree allocates tree; DefaultStarPreferences calls hub/spoke defaults).
- **Calls:** `DefaultHubPreferences`, `DefaultSpokePreferences`, `HubPreferencesTree`, `SpokePreferencesTree`.

## Control Flow Notes
**Lifecycle:**  
1. Engine calls `Enter(inNetStatePtr)` to initialize.  
2. Engine calls `Sync(topology, ΓÇª, isServer)` when joining/hosting a game ΓåÆ initializes hub (server) or spoke (client).  
3. `PacketHandler` invoked on each incoming UDP packet ΓåÆ routed to hub/spoke.  
4. Spoke-side code calls `GetNetTime`, `GetUnconfirmedActionFlagsCount`, `UpdateUnconfirmedActionFlags`, `CheckWorldUpdate` during game loop.  
5. Engine calls `UnSync(ΓÇª)` when leaving game ΓåÆ cleans up and stops hub/spoke.  

**Not inferable:** Exact tick-update and frame-render integration.

## External Dependencies
- **cseries.h**: Core platform/endianness definitions, data types.
- **StarGameProtocol.h**: Class declaration.
- **network_star.h**: Hub/spoke low-level functions (`hub_initialize`, `spoke_initialize`, etc.), message type constants, `TickBasedActionQueue` typedef.
- **TickBasedCircularQueue.h**: `WritableTickBasedCircularQueue`, `ConcreteTickBasedCircularQueue` templates.
- **player.h**: `GetRealActionQueues()` (defined elsewhere); player constants/macros.
- **interface.h**: `process_action_flags()` (defined in vbl.c).
- **InfoTree.h**: XML/INI tree configuration.
- **Defined elsewhere:** `hub_initialize`, `hub_cleanup`, `hub_received_network_packet`, `spoke_initialize`, `spoke_cleanup`, `spoke_received_network_packet`, `spoke_get_net_time`, `spoke_get_unconfirmed_flags_queue`, `spoke_get_smallest_unconfirmed_tick`, `spoke_check_world_update`, `HubParsePreferencesTree`, `SpokeParsePreferencesTree`, `DefaultHubPreferences`, `DefaultSpokePreferences`, `HubPreferencesTree`, `SpokePreferencesTree`.
