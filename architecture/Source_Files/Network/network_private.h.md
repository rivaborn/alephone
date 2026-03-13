# Source_Files/Network/network_private.h

## File Purpose
Private header for the Aleph One network subsystem containing internal packet definitions, topology structures, and service discovery classes. Intended for use only within the networking code, not by external game systems.

## Core Responsibilities
- Define network packet tags and stream packet types for inter-process communication
- Define topology structures (NetTopology, NetPlayer, NetServer) representing game network state
- Define error codes and constants for network operations
- Provide service location announcement classes for gatherer/joiner discovery
- Define chat message data structures (ClientChatInfo)
- Support migration of network protocol versions through versioned packet formats

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| NetPlayer | struct | Represents a player's network state: addresses, identifier, stream ID, connectivity status, and player info |
| NetServer | struct | Represents server network information: DSP and DDP addresses |
| NetTopology | struct | Game network topology: player list, server info, game data, next identifier counter |
| ClientChatInfo | struct | Chat message metadata: player name, color, team |
| GathererAvailableAnnouncer | class | Announces this machine as a game gatherer via SSLP service discovery |
| JoinerSeekingGathererAnnouncer | class | Searches for available gatherers via SSLP service discovery |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| GAME_PORT | macro | global | Derives game port from network_preferences |
| strNETWORK_ERRORS | constant | static | Resource ID for 132 network error strings |

## Key Functions / Methods

### GathererAvailableAnnouncer::GathererAvailableAnnouncer / ~GathererAvailableAnnouncer
- **Purpose**: Constructor/destructor; manage SSLP service instance for announcing this machine as a game gatherer
- **Side effects**: Registers/unregisters SSLP service instance on construction/destruction

### GathererAvailableAnnouncer::pump
- **Signature**: `static void pump()`
- **Purpose**: Polling update for SSLP discovery mechanism
- **Side effects**: Processes pending SSLP events

### JoinerSeekingGathererAnnouncer::JoinerSeekingGathererAnnouncer(bool shouldSeek)
- **Purpose**: Constructor; optionally start searching for gatherers via SSLP
- **Inputs**: shouldSeek flag
- **Side effects**: Registers SSLP service location callbacks if shouldSeek=true

### JoinerSeekingGathererAnnouncer::found_gatherer_callback / lost_gatherer_callback
- **Purpose**: Static callback handlers invoked when SSLP discovers or loses a gatherer service
- **Notes**: May execute in different thread than caller; provides SSLP_ServiceInstance pointer

## Control Flow Notes
This is a header-only definition file with no direct control flow. The GathererAvailableAnnouncer and JoinerSeekingGathererAnnouncer classes integrate with the SSLP (Simple Service Location Protocol) layer for network peer discovery during the netGathering/netConnecting states. The pump() pattern indicates polling-based event processing for non-threaded SSLP implementations.

## External Dependencies
- **cstypes.h**: Sized integer types (int16, uint8, uint32)
- **network.h**: Public network subsystem interface (includes game_info, player_info, network capabilities)
- **SSLP_API.h**: Service location discovery protocol (SSLP_ServiceInstance, SSLP discovery functions)
- **\<memory\>**: C++ standard library (for std::shared_ptr usage in forward-declared types)
- **Forward declarations**: CommunicationsChannel, MessageDispatcher, MessageHandler, Message (defined elsewhere)
