# Source_Files/Network/NetworkGameProtocol.h

## File Purpose
Abstract interface defining the contract for network game protocol implementations in Aleph One. Handles synchronization, packet distribution, network timing, and client-side action prediction across networked players.

## Core Responsibilities
- Define entry/exit points for network game lifecycle (Enter, Sync, UnSync)
- Manage incoming UDP packet processing
- Provide network timing information
- Track unconfirmed action flags for client-side prediction
- Check for world state updates during gameplay

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| NetworkGameProtocol | Abstract class (interface) | Base contract for all protocol implementations |

## Global / File-Static State
None.

## Key Functions / Methods

### Enter
- Signature: `virtual bool Enter(short* inNetStatePtr) = 0`
- Purpose: Initialize and enter the network game
- Inputs: Pointer to network state (output parameter)
- Outputs/Return: Boolean success flag
- Side effects: Modifies network state
- Notes: Pure virtual; implementation-specific behavior

### Sync
- Signature: `virtual bool Sync(NetTopology* inTopology, int32 inSmallestGameTick, int inLocalPlayerIndex, bool isServer) = 0`
- Purpose: Synchronize local game state with network topology
- Inputs: Network topology, game tick baseline, local player index, server flag
- Outputs/Return: Boolean success flag
- Side effects: Synchronizes game state across network
- Notes: Called during game initialization or state reconciliation

### UnSync
- Signature: `virtual bool UnSync(bool inGraceful, int32 inSmallestPostgameTick) = 0`
- Purpose: Unsynchronize and cleanly exit network game
- Inputs: Graceful flag, postgame tick baseline
- Outputs/Return: Boolean success flag
- Side effects: Releases network resources, broadcasts disconnect
- Notes: `inGraceful` determines orderly vs. abrupt shutdown

### PacketHandler
- Signature: `virtual void PacketHandler(UDPpacket& inPacket) = 0`
- Purpose: Process incoming UDP packet
- Inputs: UDP packet reference
- Outputs/Return: None
- Side effects: Updates game state, distributes packets, broadcasts topology
- Notes: Called per received packet; critical for multiplayer sync

### GetNetTime
- Signature: `virtual int32 GetNetTime() = 0`
- Purpose: Retrieve current network timestamp
- Outputs/Return: Network time value
- Notes: Used for latency compensation and action ordering

### Unconfirmed Action Flags (3 methods)
- `GetUnconfirmedActionFlagsCount()`: Returns count of pending unconfirmed actions
- `PeekUnconfirmedActionFlag(int32 offset)`: Retrieves action flag at offset without consuming
- `UpdateUnconfirmedActionFlags()`: Refreshes unconfirmed action queue
- Purpose: Support client-side prediction during server acknowledgment latency
- Notes: Actions are provisional until server confirms; enables responsive local input

### CheckWorldUpdate
- Signature: `virtual bool CheckWorldUpdate() = 0`
- Purpose: Poll for incoming world state updates
- Outputs/Return: Boolean indicating update availability
- Notes: Called per frame during gameplay

## Control Flow Notes
1. **Initialization**: `Enter()` ΓåÆ `Sync()` establishes network session
2. **Gameplay loop**: Each frame calls `PacketHandler()` (per packet), `GetNetTime()`, unconfirmed action methods, `CheckWorldUpdate()`
3. **Shutdown**: `UnSync()` gracefully terminates session
4. Used by concrete subclasses (e.g., RingGameProtocol per comments in network_private.h)

## External Dependencies
- `network_private.h`: Provides `UDPpacket`, `NetTopology`, `NetPlayer`, game/player info structures
- Concrete implementations expected to handle actual packet format, serialization, and distribution logic
