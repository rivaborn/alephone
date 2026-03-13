# Subsystem Overview

## Purpose
Implements multiplayer networking for Aleph One, enabling peer-to-peer and hub-and-spoke game topologies with real-time synchronization, metaserver integration for game discovery, and message routing across networked players.

## Key Files
| File | Role |
|------|------|
| network.cpp / network.h | Core multiplayer APIΓÇöplayer gathering/joining, connection lifecycle, state machine |
| network_messages.cpp / network_messages.h | Game protocol message serialization (join handshake, topology, map/physics distribution) |
| network_star_hub.cpp | Hub-side star topologyΓÇöcollects player actions, distributes synchronized state |
| network_star_spoke.cpp | Client-side star topologyΓÇösends actions, receives hub state, maintains timing |
| StarGameProtocol.cpp / StarGameProtocol.h | Game protocol interface adapter for star topology |
| network_games.cpp / network_games.h | Multiplayer game mode logic (scoring, rankings, game-over detection) |
| network_metaserver.cpp / network_metaserver.h | Metaserver clientΓÇölogin, game listing, chat relay, player list sync |
| metaserver_dialogs.cpp / metaserver_dialogs.h | UI layer for metaserver (game discovery, chat, join) |
| network_dialogs.cpp / network_dialogs.h | UI dialogs for game gathering, joining, setup configuration |
| ConnectPool.cpp / ConnectPool.h | Async TCP connection pooling with background threads for DNS resolution |
| NetworkInterface.cpp / NetworkInterface.h | ASIO-based socket abstraction (TCP/UDP, IP address handling) |
| network_udp.cpp | UDP socket lifecycle and async receive thread |
| SSLP_limited.cpp / SSLP_API.h | LAN service discovery (Simple Service Location Protocol) |
| Pinger.cpp / Pinger.h | ICMP ping measurement for latency monitoring |
| StandaloneHub/ | Dedicated network hub server (gatherer relay) |
| network_private.h | Internal packet definitions, topology structures, service discovery |
| HTTP.cpp / HTTP.h | libcurl-based HTTP client for metaserver and update checks |
| Update.cpp / Update.h | Background update version checking |
| PortForward.cpp / PortForward.h | UPnP port forwarding for NAT traversal |

## Core Responsibilities
- **Connection Management**: Accept player connections, negotiate capabilities, manage client lifecycle (connecting ΓåÆ connected ΓåÆ in-game)
- **Message Routing**: Deserialize incoming network messages, dispatch to handlers, queue outgoing messages (chat, topology, game state)
- **Topology Distribution**: Maintain and broadcast player list and game state to all connected players
- **Game Data Synchronization**: Compress and stream map data, physics scripts, and embedded Lua to joiners; accumulate received data on joiner side
- **Star Protocol Implementation**: Hub collects action flags from all spokes, calculates timing offsets, synthesizes flags for lagging players, broadcasts synchronized frames
- **Game Discovery**: Advertise local games to metaserver and LAN via SSLP; browse and join remote games
- **Player Authentication**: Handle metaserver login (plaintext, XOR, HTTPS key exchange) and maintain authenticated session
- **Network Diagnostics**: Measure per-player latency and jitter via ping; track packet loss and statistics
- **Non-Blocking Connectivity**: Spawn background threads for async DNS resolution and TCP handshake to avoid blocking game loop

## Key Interfaces & Data Flow
**What this subsystem exposes:**
- `network.h` API: `NetEnter()`, `NetExit()`, `NetGather()`, `NetJoin()`, `NetSync()`, `NetChangeMap()`, `NetStart()`, `NetProcessNetMessage()`
- Callbacks for chat (`ChatCallbacks`), gathering (`GatherCallbacks`), metaserver notifications
- Diagnostic APIs: `NetGetPlayerData()`, latency/jitter queries, network statistics

**What it consumes:**
- Game state: `dynamic_world`, map polygons, player data, physics, embedded Lua scripts (via `interface.h`)
- Preferences: network settings (gather port, remote hub address, login credentials, muting rules)
- UI: dialog results, player selection, chat input (via `network_dialogs.h`, `metaserver_dialogs.h`)
- System: time (ticks via `machine_tick_count()`), filesystem (WAD loading), preferences persistence

## Runtime Role
- **Initialization** (`NetEnter`): Establish network session, spawn connection listener or initiate remote connection
- **Pre-Game** (`NetGather` / `NetJoin`): Accept/connect joiners, negotiate capabilities, exchange topology and game data
- **Per-Frame** (`NetSync`): Receive and dispatch incoming messages; for hub spokes, retrieve and buffer action flags; for gatherer, broadcast synchronized state
- **Shutdown** (`NetExit`): Close sockets, terminate threads, release resources

## Notable Implementation Details
- **Star topology synchronization**: Hub maintains separate tick-based circular action queues per player; uses windowed Nth-element statistics to calculate per-player timing offsets; synthesizes action flags for stalled players to reduce bandwidth
- **Hub packet structure**: Includes acknowledgment tracking, address mapping for NAT (responds to any source address from a player's IP), and per-player timing adjustment values
- **Non-blocking connections**: `ConnectPool` spawns SDL background threads for each connection attempt (max 20), resolves hostnames asynchronously, returns `CommunicationsChannel` on success
- **Message serialization**: zlib compression for large payloads (maps, physics, Lua scripts); big-endian binary format; capability version negotiation via `Capabilities` map
- **Metaserver integration**: Supports guest login (no authentication), plaintext password, XOR encryption, and HTTPS key derivation; relays chat/private messages and player state changes; filters ignored players at handler level
- **SSLP service discovery**: Pump-based design (non-threaded); broadcasts FIND packets, processes HAVE/LOST responses; maintains linked-list timeout garbage collection for stale entries; game instances advertise with map/scenario/difficulty metadata
- **Standalone hub**: Acts as gatherer relay for remote topology; accepts connection from hosting gatherer process via TCP, stores game data messages, coordinates joiner connections
- **Network capabilities**: Enumerated feature flags (gameworld version, star protocol, Lua support, zipped data, network stats, rugby mode) allow version negotiation between clients
