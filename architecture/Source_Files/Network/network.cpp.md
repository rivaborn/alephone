# Source_Files/Network/network.cpp

## File Purpose

Core multiplayer networking implementation for Aleph One (Marathon engine). Manages server/client architecture for networked games, including player connection lifecycle, topology distribution, message routing, game data synchronization (maps, physics, Lua scripts), and chat relay. Supports both traditional network gathering and remote hub coordination.

## Core Responsibilities

- **Connection Management**: Establish server socket, accept joiners, manage client lifecycle (connect, disconnect, drop)
- **Client State Machine**: Track each joiner through _connecting ΓåÆ _awaiting_capabilities ΓåÆ _connected ΓåÆ _awaiting_map ΓåÆ _ingame
- **Topology Distribution**: Maintain and broadcast player list, game state to all players
- **Message Routing**: Dispatch network messages to handlers (chat, capabilities, map, physics, topology)
- **Game Data Distribution**: Compress and stream map/physics/Lua scripts to joiners; accumulate received data on joiner side
- **Capability Negotiation**: Check version/feature compatibility (gameworld, Star protocol, Lua, Rugby)
- **Chat Relay**: Broadcast player messages during pre-game and in-game phases
- **Network Statistics**: Collect and distribute per-player latency/packet stats
- **Saved Game Support**: Resume networked games with preserved topology and player data
- **Remote Hub Coordination**: Forward data to remote dedicated server instead of local gatherer

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| Client | class | Connected player state, message handlers, _connecting/_connected/_awaiting_map/_ingame transitions |
| NetTopology | struct | Game state, player array, server address, game/player data |
| NetPlayer | struct | Player identifier, network addresses (DSP/DDP), stream ID, net_dead flag |
| ClientChatInfo | struct | Name, color, team for pre-game chat display |
| prospective_joiner_info | struct | stream_id, name, color, team, gathering flag |
| Capabilities | class | Feature flags (kGameworld, kStar, kLua, kZippedData, kRugby, kNetworkStats) |
| NetworkStats | struct | Latency, packet loss, ping stats per player |
| CommunicationsChannel | class | Abstract network I/O interface (enqueue/pump/dispatch messages) |
| MessageDispatcher | class | Route messages by type to registered handler objects |
| MessageInflater | class | Decompress message data |
| StarGameProtocol | class | Game protocol implementation managing ticks and action flags |
| NonblockingConnect | class | Async connection attempts with Connected/ResolutionFailed/ConnectFailed states |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| localPlayerIndex | short | static | Index of local player in topology (0 for server, varies for joiner) |
| localPlayerIdentifier | short | static | Network ID assigned to local player |
| topology | NetTopology* | static | Current game topology (players, addresses, game data) |
| netState | short | static | Master state machine (netUninitialized, netGathering, netJoining, netWaiting, netActive, netDown) |
| sCurrentGameProtocol | StarGameProtocol | static | Active game protocol instance |
| connections_to_clients | std::map<int, Client*> | static | Map stream_idΓåÆClient for all connected joiners (server-side) |
| connection_to_server | std::unique_ptr<CommunicationsChannel> | static | Single connection to gatherer (joiner-side) |
| next_stream_id | int | static | Counter for assigning unique stream IDs to joiners |
| server_nbc | NonblockingConnect* | static | In-progress connection attempt during joining |
| handlerState | short | static | Join-phase state machine (netAwaitingHello, netJoining, netWaiting) |
| handlerMapBuffer | byte* | static | Accumulated map data received from server |
| handlerPhysicsBuffer | std::vector<byte> | static | Accumulated physics model data |
| handlerLuaBuffer | std::vector<byte> | static | Accumulated Lua script data |
| deferred_script | std::vector<byte> | static | Lua script to distribute to all joiners |
| client_chat_info | std::map<int, ClientChatInfo*> | static | Pre-game chat info by stream_id |
| sNetworkStats | std::vector<NetworkStats> | static | Latest network stats snapshot from server |
| sIgnoredPlayers | std::set<int> | static | Player indices on local ignore list |
| resuming_saved_game | bool | static | Whether game is resuming from save (vs. new game) |
| use_remote_hub | bool | static | Using remote hub instead of local server |

## Key Functions / Methods

### Client::Client(std::shared_ptr<CommunicationsChannel>)
- Signature: `Client(std::shared_ptr<CommunicationsChannel> inChannel)`
- Purpose: Construct and initialize a new client connection with message handlers
- Inputs: CommunicationsChannel shared_ptr for this client
- Outputs/Return: Client object with message dispatcher ready
- Side effects: Allocates 8 message handler objects; registers them with dispatcher
- Calls: newMessageHandlerMethod (template), mDispatcher->setHandlerForType
- Notes: Sets up handlers for JoinerInfoMessage, CapabilitiesMessage, AcceptJoinMessage, ChatMessage, ChangeColorsMessage, RemoteHubCommandMessage, RemoteHubHostConnectMessage

### Client::drop()
- Signature: `void drop()`
- Purpose: Disconnect and clean up a client, removing from topology if gathered
- Inputs: None (uses member channel, state)
- Outputs/Return: None
- Side effects: Modifies topology if _awaiting_map state; sends NetDistributeTopology(tagDROPPED_PLAYER); calls gatherCallbacks; announces to metaserver; deletes ClientChatInfo
- Calls: gatherCallbacks->JoinedPlayerDropped/JoiningPlayerDropped, gMetaserverClient->announcePlayersInGame, NetUpdateTopology, NetDistributeTopology
- Notes: Behavior differs for _connected vs _awaiting_map states; removes from connections_to_clients

### Client::capabilities_indicate_player_is_gatherable(bool warn_joiner)
- Signature: `bool capabilities_indicate_player_is_gatherable(bool warn_joiner)`
- Purpose: Determine if joiner's declared capabilities meet gatherer requirements
- Inputs: warn_joiner - send ServerWarningMessage to joiner if incompatible
- Outputs/Return: true if joiner can participate, false if version mismatch or feature missing
- Side effects: Sends ServerWarningMessage to channel if incompatible and warn_joiner=true
- Calls: shapes_file_is_m1, channel->enqueueOutgoingMessage
- Notes: Checks kGameworld version, kGameworldM1, kStar, kLua, kRugby; kGatherable=0 suppresses warning

### Client::handleJoinerInfoMessage(JoinerInfoMessage*, CommunicationsChannel*)
- Signature: `void handleJoinerInfoMessage(JoinerInfoMessage*, CommunicationsChannel*)`
- Purpose: Process joiner's initial name/color/team announcement
- Inputs: JoinerInfoMessage with player name and preferences
- Outputs/Return: None
- Side effects: Stores name in member; creates ClientChatInfo; broadcasts ClientInfoMessage(kAdd) to other clients; sends CapabilitiesMessage; transitions to _awaiting_capabilities
- Calls: CapabilitiesMessage, ClientInfoMessage
- Notes: Only valid when netState==netGathering; version mismatch sets _disconnect state

### Client::handleCapabilitiesMessage(CapabilitiesMessage*, CommunicationsChannel*)
- Signature: `void handleCapabilitiesMessage(CapabilitiesMessage*, CommunicationsChannel*)`
- Purpose: Receive joiner capabilities and decide if player is gatherable
- Inputs: CapabilitiesMessage from joiner
- Outputs/Return: None
- Side effects: Copies capabilities; calls capabilities_indicate_player_is_gatherable; broadcasts ClientInfoMessage(kAdd) for all known clients to new joiner; transitions to _connected_but_not_yet_shown or _ungatherable
- Calls: capabilities_indicate_player_is_gatherable, ClientInfoMessage, NetGatherPlayer (on standalone hub)
- Notes: Critical compatibility gate; gathers player immediately on standalone hub

### Client::handleAcceptJoinMessage(AcceptJoinMessage*, CommunicationsChannel*)
- Signature: `void handleAcceptJoinMessage(AcceptJoinMessage*, CommunicationsChannel*)`
- Purpose: Add accepted joiner to topology; broadcast topology update
- Inputs: AcceptJoinMessage with player data and identifier
- Outputs/Return: None
- Side effects: Adds player to topology array; increments player_count; sends GameSessionMessage; calls check_player callback; calls gatherCallbacks->JoinSucceeded; announces to metaserver; distributes updated topology; transitions to _awaiting_map
- Calls: NetUpdateTopology, NetDistributeTopology(tagNEW_PLAYER), check_player callback, gatherCallbacks->JoinSucceeded
- Notes: One of the critical state transitions for adding a player

### NetProcessNewJoiner(std::shared_ptr<CommunicationsChannel>)
- Signature: `bool NetProcessNewJoiner(std::shared_ptr<CommunicationsChannel> new_joiner)`
- Purpose: Set up incoming connection as a Client object and send HelloMessage
- Inputs: New CommunicationsChannel from server accept()
- Outputs/Return: true if processed, false if null input
- Side effects: Creates Client; adds to connections_to_clients[next_stream_id]; increments next_stream_id; sends HelloMessage
- Calls: Client constructor, HelloMessage
- Notes: Server-side only; called from NetCheckForNewJoiner

### NetCheckForNewJoiner(prospective_joiner_info&, CommunicationsChannelFactory*, bool)
- Signature: `bool NetCheckForNewJoiner(prospective_joiner_info&, CommunicationsChannelFactory*, bool process_new_joiners)`
- Purpose: Poll for new incoming connections; pump and dispatch all existing clients
- Inputs: Output info struct, optional server factory override, process_new_joiners flag
- Outputs/Return: true if a client transitioned to _connected_but_not_yet_shown
- Side effects: Calls NetProcessNewJoiner for new connections; pumps/dispatches all client channels; removes disconnected clients; fills prospective_joiner_info on first gatherability transition
- Calls: actual_server->newIncomingConnection, NetProcessNewJoiner, channel->pump, channel->dispatchIncomingMessages, Client::drop
- Notes: Called each frame during gathering; filters for can_pregame_chat() state transitions

### NetUpdateJoinState(void)
- Signature: `short NetUpdateJoinState(void)`
- Purpose: Drive state machine for joining: netConnecting ΓåÆ netJoining ΓåÆ netWaiting
- Inputs: None (uses module statics: netState, server_nbc, connection_to_server)
- Outputs/Return: New network state (or internal state like netPlayerAdded)
- Side effects: Attempts non-blocking connections via ConnectPool; sets handlerState; transitions netState; shows alerts on error
- Calls: ConnectPool::instance()->connect, server_nbc->status(), connection_to_server->setMessageInflater/setMessageHandler, alert_user
- Notes: Implements 5-second retry logic for connection attempts; handles resolution failures and connect failures separately

### NetGatherPlayer(const prospective_joiner_info&, CheckPlayerProcPtr check_player)
- Signature: `int NetGatherPlayer(const prospective_joiner_info&, CheckPlayerProcPtr check_player)`
- Purpose: Accept a joiner and send JoinPlayerMessage with assigned identifier
- Inputs: prospective_joiner_info (stream_id, name), optional check_player callback
- Outputs/Return: kGatherPlayerSuccessful constant
- Side effects: Stores check_player callback in Client::check_player static; sends JoinPlayerMessage to client; transitions client to _awaiting_accept_join
- Calls: JoinPlayerMessage
- Notes: Callback invoked later by handleAcceptJoinMessage for color reassignment

### NetDistributeGameDataToAllPlayers(byte*, int32, bool, CommunicationsChannel*)
- Signature: `OSErr NetDistributeGameDataToAllPlayers(byte* wad_buffer, int32 wad_length, bool do_physics, CommunicationsChannel* remote_hub)`
- Purpose: Send map, physics model, and Lua script to all players; manage compression
- Inputs: WAD buffer, length, physics flag, optional remote_hub channel
- Outputs/Return: OSErr (0=noErr)
- Side effects: Creates two client lists (zip-capable, zip-incapable); sends ZippedMap/Map, ZippedPhysics/Physics, ZippedLua/Lua, EndGameData; processes physics locally; calls process_network_physics_model; shows progress dialog
- Calls: std::bind with enqueueOutgoingMessage, multipleFlushOutgoingMessages, process_network_physics_model, LoadLuaScript
- Notes: Deflates compressed messages once for reuse; separates clients by capability; uses std::for_each with bind for broadcast

### NetReceiveGameData(bool do_physics)
- Signature: `byte* NetReceiveGameData(bool do_physics)`
- Purpose: Joiner waits for and accumulates map/physics/script from server
- Inputs: do_physics flag
- Outputs/Return: Pointer to map buffer (caller must free via process_net_map_data)
- Side effects: Opens progress dialog; calls receiveSpecificMessage for EndGameDataMessage; processes accumulated buffers; calls process_network_physics_model and LoadLuaScript
- Calls: connection_to_server->receiveSpecificMessage, process_network_physics_model, LoadLuaScript, draw_progress_bar, alert_user
- Notes: Handlers (handleMapMessage, handlePhysicsMessage, handleLuaMessage) populate static buffers; waits 60 seconds with 30-second timeout

### NetProcessMessagesInGame(void)
- Signature: `void NetProcessMessagesInGame()`
- Purpose: Per-frame message processing during active gameplay
- Inputs: None (uses connections_to_clients or connection_to_server)
- Outputs/Return: None
- Side effects: Pumps and dispatches all channels; sends NetworkStatsMessage periodically from server; pumps metaserver
- Calls: channel->pump, channel->dispatchIncomingMessages, gMetaserverClient->pump
- Notes: Different code path for server (all connections) vs joiner (single server connection); stats sent every MACHINE_TICKS_PER_SECOND

### NetChangeMap(struct entry_point*)
- Signature: `bool NetChangeMap(struct entry_point* entry)`
- Purpose: Load new level; distribute map to all players if server
- Inputs: entry_point for target level
- Outputs/Return: true if map successfully loaded
- Side effects: Gets map/physics from appropriate source; calls NetDistributeGameDataToAllPlayers or NetReceiveGameData; processes map; shows progress
- Calls: get_map_for_net_transfer, get_net_map_data_length, NetDistributeGameDataToAllPlayers, NetReceiveGameData, process_net_map_data
- Notes: Server path vs joiner path diverge; supports StandaloneHub and remote_hub; clears sNetworkStats on server

### reassign_player_colors(short player_index, short num_players)
- Signature: `void reassign_player_colors(short player_index, short num_players)`
- Purpose: Ensure all players have unique colors, respecting team assignments
- Inputs: player_index (unused), num_players to assign
- Outputs/Return: None (modifies topology->players in-place)
- Side effects: Updates player_info.color and player_info.team for all players
- Calls: NetGetPlayerData, NetGetGameData
- Notes: Two modes: _force_unique_teams (each player gets unique color) vs team-based (unique colors within each team)

### match_starts_with_existing_players(player_start_data*, short*)
- Signature: `void match_starts_with_existing_players(player_start_data* ioStartArray, short* ioStartCount)`
- Purpose: Align player starts from saved game with current player count
- Inputs: Start array, player count
- Outputs/Return: None (modifies in-place)
- Side effects: Reorders starts to match current players by name matching; creates new starts for new players; assigns remaining starts to player slots
- Calls: get_player_data, strcmp
- Notes: Handles cases: players added, players removed, players changed

### handleTopologyMessage(TopologyMessage*, CommunicationsChannel*) [static]
- Signature: `static void handleTopologyMessage(TopologyMessage*, CommunicationsChannel*)`
- Purpose: Joiner receives topology update during netWaiting state
- Inputs: TopologyMessage with current player list
- Outputs/Return: None
- Side effects: Replaces local topology; processes game start; may call match_starts_with_existing_players and NetChangeMap; transitions netState based on tag
- Calls: match_starts_with_existing_players, NetChangeMap, LoadLuaScript
- Notes: Complex branching on topology->tag (tagRESUME_GAME, tagPLAYERS, etc.); critical for game initialization on joiner

### handleNetworkChatMessage(NetworkChatMessage*, CommunicationsChannel*) [static, joiner-side]
- Signature: `static void handleNetworkChatMessage(NetworkChatMessage*, CommunicationsChannel*)`
- Purpose: Display received chat during joining or in-game phases
- Inputs: NetworkChatMessage from server
- Outputs/Return: None
- Side effects: Calls chatCallbacks->ReceivedMessageFromPlayer
- Calls: chatCallbacks->ReceivedMessageFromPlayer, logAnomaly
- Notes: Looks up player name from topology or client_chat_info; checks ignore list in in-game phase

### Client::handleChatMessage(NetworkChatMessage*, CommunicationsChannel*) [member, server-side]
- Signature: `void handleChatMessage(NetworkChatMessage*, CommunicationsChannel*)`
- Purpose: Relay chat message from one client to others (server-side broadcast)
- Inputs: NetworkChatMessage from sending client
- Outputs/Return: None
- Side effects: Broadcasts message to clients in same state (pre-game or in-game); displays locally if chatCallbacks set
- Calls: connections_to_clients iteration, channel->enqueueOutgoingMessage, chatCallbacks->ReceivedMessageFromPlayer
- Notes: Different relay logic for in-game (broadcast to _ingame clients) vs pre-game (broadcast to can_pregame_chat clients)

### NetAllowCrosshair(), NetAllowTunnelVision(), NetAllowBehindview(), NetAllowCarnageMessages(), NetAllowSavingLevel(), NetAllowOverlayMap()
- Signature: `bool NetAllow*()`
- Purpose: Determine if UI feature is allowed in current network context
- Inputs: None (queries dynamic_world, netState)
- Outputs/Return: true if feature enabled, false if disabled
- Side effects: None
- Calls: None
- Notes: Single-player always returns true; multiplayer checks game_information.cheat_flags bits

## Control Flow Notes

**Server (Gatherer) Flow:**
1. NetInitialize() - topology initialized with local player at index 0
2. NetGather() repeatedly called from main loop
3. Each frame: NetCheckForNewJoiner() polls accept(), processes all clients, returns first client to transition to _connected_but_not_yet_shown
4. Client flow: HelloMessage ΓåÆ JoinerInfoMessage ΓåÆ CapabilitiesMessage ΓåÆ (gather decision) ΓåÆ JoinPlayerMessage ΓåÆ AcceptJoinMessage ΓåÆ awaiting_map
5. NetChangeMap() - server distributes map via NetDistributeGameDataToAllPlayers(), clients transition to _ingame
6. NetProcessMessagesInGame() pumps all connections each frame

**Joiner Flow:**
1. NetUpdateJoinState() drives connection loop: attempts connect every 5 seconds via ConnectPool
2. On connect: setMessageInflater and setMessageHandler for join dispatcher
3. Join dispatcher routes to handlers: handleHelloMessage ΓåÆ handleCapabilitiesMessage ΓåÆ handleJoinPlayerMessage ΓåÆ handleTopologyMessage
4. Each message handler updates handlerState module static
5. handleTopologyMessage with tagRESUME_GAME or tagPLAYERS triggers NetChangeMap() to receive game data
6. NetReceiveGameData() waits for map/physics/lua, then processes
7. NetProcessMessagesInGame() pumps single connection during gameplay

**Map/Physics Distribution (Server ΓåÆ Joiner):**
- Server: NetChangeMap() calls NetDistributeGameDataToAllPlayers()
  - Separates clients into zip-capable and zip-incapable lists
  - For each: creates ZippedMapMessage or MapMessage, deflates once, broadcasts via enqueueOutgoingMessage
  - Flushes all channels together, then updates Client states to _ingame
- Joiner: NetChangeMap() calls NetReceiveGameData()
  - Waits for EndGameDataMessage with 60s timeout, 30s per message
  - Handlers populate handlerMapBuffer, handlerPhysicsBuffer, handlerLuaBuffer
  - Processes buffers: process_network_physics_model, LoadLuaScript, process_net_map_data

**Player Drop During Gathering:**
- Client disconnects or calls drop()
- NetCheckForNewJoiner() detects disconnection, calls Client::drop()
- drop() updates topology, sends NetDistributeTopology(tagDROPPED_PLAYER), calls gatherCallbacks->JoinedPlayerDropped

**Player Drop During Game:**
- Server removes from connections_to_clients
- Sends ClientInfoMessage(stream_id, kRemove) to all other clients
- Other servers see drop via client_chat_info erase

## External Dependencies

**Major includes:**
- `map.h` - TICKS_PER_SECOND, entry_point, game_data, player_info, dynamic_world
- `interface.h` - get_map_for_net_transfer(), process_net_map_data(), get_net_map_data_length()
- `preferences.h` - network_preferences, player_preferences, environment_preferences
- `player.h` - player structures
- `MessageDispatcher.h`, `MessageHandler.h` - message routing system
- `network_messages.h` - Message subclasses (HelloMessage, TopologyMessage, etc.)
- `StarGameProtocol.h` - game protocol
- `ConnectPool.h` - non-blocking connection pooling
- `progress.h` - progress dialog UI

**Defined elsewhere:**
- `dynamic_world` (world.cpp) - game state singleton
- `get_game_state()`, `get_player_data()` - game accessors
- `screen_printf()`, `alert_user()` - UI output (alerts.cpp)
- `shapes_file_is_m1()` - shape compatibility check
- `gMetaserverClient` - metaserver singleton
- `gatherCallbacks`, `chatCallbacks` - callback function objects
- `StandaloneHub::Instance()` - remote hub (conditional A1_NETWORK_STANDALONE_HUB)

**Conditional compilation:**
- If DISABLE_NETWORKING defined, network_dummy.cpp included instead, providing stubs
