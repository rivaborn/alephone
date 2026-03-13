# Source_Files/Network/ConnectPool.cpp
## File Purpose
Implements a non-blocking TCP connection pool for the Aleph One game engine. Manages up to 20 concurrent asynchronous connection attempts, each running on a background SDL thread to avoid blocking game logic during DNS resolution and TCP handshakes.

## Core Responsibilities
- Spawn and manage background threads for non-blocking TCP connections
- Resolve hostnames asynchronously using the network interface
- Maintain a fixed-size pool of reusable connection slots (kPoolSize = 20)
- Track connection state (Connecting, Connected, ResolutionFailed, ConnectFailed)
- Provide lazy cleanup of completed connections
- Release CommunicationsChannel objects to callers on successful connection

## External Dependencies
- **SDL:** `SDL_CreateThread`, `SDL_WaitThread` (thread lifecycle)
- **CommunicationsChannel:** custom class for TCP I/O and connection management (defined elsewhere)
- **NetworkInterface:** `NetGetNetworkInterface()->resolve_address()` for DNS resolution (defined elsewhere)
- **Standard library:** `<string>`, `<memory>`, `<utility>`
- **Type definitions:** `uint16`, `IPaddress` (from cseries.h)

# Source_Files/Network/ConnectPool.h
## File Purpose
Provides a singleton pool for managing non-blocking outbound TCP connections in a multi-threaded context. Allows asynchronous connection establishment with status polling and connection reuse via pooling.

## Core Responsibilities
- Manage individual non-blocking TCP connection attempts (`NonblockingConnect`) with per-connection status tracking
- Provide thread-safe asynchronous connection establishment (address resolution + TCP connect)
- Maintain a singleton pool of reusable `NonblockingConnect` instances (max 20)
- Track connection lifecycle (Connecting ΓåÆ Connected or failed states)
- Release completed connections as `CommunicationsChannel` pointers for use by callers

## External Dependencies
- **"cseries.h":** Platform abstraction, type definitions (`uint16`).
- **"CommunicationsChannel.h":** `CommunicationsChannel` class and `IPaddress` type.
- **SDL2:** `SDL_Thread` for background connection threads.
- **Standard library:** `<string>`, `<memory>` (for `std::unique_ptr`).

# Source_Files/Network/HTTP.cpp
## File Purpose
Implements a lightweight HTTP client wrapper around libcurl for GET and POST requests. Provides conditional support for HTTPS with configurable certificate verification, used by the game engine for network communication (e.g., metaserver interaction).

## Core Responsibilities
- Initialize libcurl at engine startup via `Init()`
- Execute HTTP GET requests and capture response bodies
- Execute HTTP POST requests with URL-encoded form parameters
- Handle response streaming via a write callback mechanism
- Support HTTPS with verification control from network preferences
- Degrade gracefully when libcurl is unavailable (stub implementations)

## External Dependencies
- **libcurl:** `curl/curl.h`, `curl/easy.h` ΓÇö HTTP client library; used for all network operations.
- **Engine utilities:** `cseries.h` ΓÇö core engine types and macros.
- **Logging:** `Logging.h` ΓÇö `logError()` macro for diagnostics.
- **Preferences:** `preferences.h` ΓÇö `network_preferences` global for `verify_https` flag.
- **HTTP header:** `HTTP.h` ΓÇö class definition and typedef for `parameter_map`.

# Source_Files/Network/HTTP.h
## File Purpose
Defines a lightweight HTTP client class for making GET and POST requests. Wraps underlying HTTP library functionality (likely libcurl) to provide simple request/response handling for game network communication.

## Core Responsibilities
- Provide HTTP GET and POST request methods
- Manage HTTP response data accumulation and retrieval
- Initialize HTTP library state at startup
- Abstract HTTP library details from game code

## External Dependencies
- `<map>` ΓÇö std::map for parameter storage
- `<string>` ΓÇö std::string for URLs and responses
- **libcurl** (inferred) ΓÇö HTTP library; WriteCallback signature and static Init suggest curl usage

# Source_Files/Network/Metaserver/metaserver_dialogs.cpp
## File Purpose

Implements the metaserver client UI dialog for Aleph One. Provides the presentation layer for game discovery, joining, and in-game chat. Handles user interactions such as selecting games/players, sending messages, and joining hosted games discovered via the metaserver.

## Core Responsibilities

- **Game Discovery UI**: Displays available games and players in a room; manages game/player selection state
- **Dialog Lifecycle**: Runs modal dialog with widget callbacks; produces a join address on successful game selection
- **Game Announcement**: Packages local game info (map, difficulty, options) and announces to metaserver
- **Chat Integration**: Bridges metaserver notifications (join/leave, messages) to global chat history; plays UI sounds
- **Update Checking**: Prompts user if a new build is available before connecting (non-Steam only)
- **Notification Routing**: Adapter pattern to receive and process metaserver events (player list, game list, chat, disconnection)

## External Dependencies

- **Network & Metaserver:** `MetaserverClient`, `MetaserverPlayerInfo`, `GameListMessage::GameListEntry`, `IPaddress` (network_metaserver.h)
- **UI Framework:** `dialog`, `w_title`, `w_spacer`, `w_static_text`, `w_hyperlink`, `w_button`, widget classes from implied header (csdialogs.h via cseries.h)
- **Game State:** `Scenario::instance()`, `game_info` struct, physics/Lua detection (`level_has_embedded_physics_lua`)
- **Preferences:** `player_preferences`, `network_preferences`, `environment_preferences` (preferences.h)
- **Update System:** `Update::instance()`, update check dialogs (Update.h, progress.h)
- **Audio:** `PlayInterfaceButtonSound()` (SoundManager.h)
- **Chat Widget:** `ColorfulChatWidget`, `ChatHistory`, `EditTextWidget`, `PlayerListWidget`, `GameListWidget`, `ButtonWidget` (shared_widgets.h)

Not defined here: `MetaserverClient` internals, widget implementations, chat history storage.

# Source_Files/Network/Metaserver/metaserver_dialogs.h
## File Purpose
Defines the UI layer for metaserver client interactions in Aleph One. Declares abstract base classes and concrete UI adapters for connecting to the metaserver, browsing games/players, and handling real-time chat and notifications.

## Core Responsibilities
- Define the `MetaserverClientUi` abstract factory for platform-specific UI implementations
- Implement observer pattern adapters (`GlobalMetaserverChatNotificationAdapter`) to receive metaserver events
- Manage UI state for player/game lists, chat entry, and interactive buttons
- Handle user interactions (join game, select player, send chat, mute)
- Coordinate game announcements via `GameAvailableMetaserverAnnouncer`

## External Dependencies
- **`network_metaserver.h`** ΓÇö `MetaserverClient`, `MetaserverClient::NotificationAdapter`, `MetaserverPlayerInfo`, `GameListMessage::GameListEntry`, `RemoteHubServerDescription`, `RoomDescription`, etc.
- **`shared_widgets.h`** ΓÇö `PlayerListWidget`, `GameListWidget`, `EditTextWidget`, `ColorfulChatWidget`, `ButtonWidget`
- **Forward declares:** `game_info` (defined elsewhere), `IPaddress` (defined elsewhere)
- **Implied:** Global instance `gMetaserverClient` referenced by `GameAvailableMetaserverAnnouncer`

# Source_Files/Network/Metaserver/metaserver_messages.cpp
## File Purpose
Implements TCPMess message serialization/deserialization for metaserver protocol communication. Provides encoding and decoding of client-server messages for login, game listings, player lists, and chat in the Marathon/Aleph One game.

## Core Responsibilities
- Serialize (deflate) and deserialize (inflate) metaserver protocol messages
- Manage player color, state, and auxiliary data encoding/decoding
- Parse and generate game descriptions with map/scenario metadata
- Handle room and game list message formats
- Support bidirectional chat and private message serialization
- Provide stream-based I/O helpers for padded strings and binary data

## External Dependencies
**Notable includes:**
- `Message.h`, `MessageDispatcher.h`, `MessageHandler.h`, `MessageInflater.h` ΓÇö message dispatch framework
- `AStream.h` ΓÇö serialization streams (`AIStream`, `AOStream`)
- `preferences.h` ΓÇö global `network_preferences`, `player_preferences`
- `shell.h` ΓÇö `get_player_color()`
- `network_dialogs.h` ΓÇö game type constants
- `TextStrings.h` ΓÇö `TS_GetCString()` for localized strings
- `map.h` ΓÇö `TICKS_PER_SECOND` constant
- `<boost/algorithm/string/predicate.hpp>` ΓÇö `ends_with()`

**Defined elsewhere (used but not declared here):**
- `RGBColor`, `rgb_color` ΓÇö color types from cseries.h
- `IPaddress` ΓÇö network address type (supports `set_address()`, `set_port()`)
- `Scenario::instance()` ΓÇö singleton for scenario/map metadata
- `_get_player_color()` ΓÇö platform-specific color lookup
- `machine_tick_count()` ΓÇö system tick timer
- `TS_GetCString()` ΓÇö text string resource lookup
- Global preferences objects (`network_preferences`, `player_preferences`)

# Source_Files/Network/Metaserver/metaserver_messages.h
## File Purpose
Defines TCPMess message types and serializable message classes for metaserver clientΓÇôserver communication. Covers login/authentication, game/player/room listings, chat, private messaging, and remote hub discovery in the Aleph One game engine.

## Core Responsibilities
- Enumerate message type constants (kSERVER_*, kCLIENT_*, kBOTH_*)
- Define serializable message classes inheriting from `SmallMessageHelper` or template `DatalessMessage`
- Implement serialization/deserialization via `reallyInflateFrom()` and `reallyDeflateTo()` methods
- Encapsulate game metadata, player profiles, room/server descriptions, and chat/private message content
- Support authentication flow (salt exchange, handoff tokens)
- Enable game listing, player roster updates, and dynamic room/remote-hub discovery

## External Dependencies

- **Message.h:** `SmallMessageHelper`, `DatalessMessage<>`, `Message` base class
- **AStream.h:** `AIStream`, `AOStream` (serialization stream classes, including endian variants)
- **Scenario.h:** `Scenario::instance()` (singleton for map/scenario metadata)
- **network.h:** `kNetworkSetupProtocolID` constant, `IPaddress` type
- **std::string, std::vector:** Standard library containers
- **boost::algorithm::to_lower_copy():** Case conversion utility
- **machine_tick_count():** External function (SDL ticks); defined elsewhere
- **uint8, uint16, uint32, int16, int32:** Typedef'd integer types (cstypes.h)

# Source_Files/Network/Metaserver/network_metaserver.cpp
## File Purpose
Implements the `MetaserverClient` class, a client for connecting to a game metaserver (Aleph One). Handles authentication, room/player/game listing, chat messaging, and game lifecycle announcements. Core networking logic for multiplayer game discovery and communication.

## Core Responsibilities
- Establish and maintain TCP connections to metaserver and room servers
- Handle authentication (supports plaintext, XOR encryption, HTTPS key exchange)
- Process incoming messages (chat, broadcast, player list updates, game list, etc.)
- Queue outgoing messages (chat, game announcements, status updates)
- Manage player ignore lists with filtering at message-handler level
- Announce games to metaserver and track player count / game state changes
- Ping game servers asynchronously to measure latency
- Route notifications to UI layer via `NotificationAdapter` callback interface

## External Dependencies
- **cseries.h**: Base types, platform macros
- **network_metaserver.h**: Class declaration, nested types
- **Message*.h** (4 files): Message framework (inflate/deflate, handlers, dispatchers)
- **preferences.h**: `network_preferences` global (mute guests, login creds)
- **alephversion.h**: Version constants, metaserver URLs, HTTPS login endpoint
- **HTTP.h**: `HTTPClient` for HTTPS key derivation
- **Logging.h**: `logAnomaly()` function
- **boost/algorithm/string/predicate.hpp**: `starts_with()` for guest name filtering
- **Standard library**: `<string>`, `<iostream>`, `<algorithm>`, `<iterator>`
- **Pinger service** (defined elsewhere): `NetRemovePinger()`, `NetCreatePinger()`, `NetGetPinger()`

# Source_Files/Network/Metaserver/network_metaserver.h
## File Purpose
Header for the metaserver client that manages connection to a lobby/matchmaking server. Handles login, player/game list synchronization, chat, and game announcements for the Aleph One game engine.

## Core Responsibilities
- Establish and maintain authenticated connection to metaserver
- Synchronize dynamic lists (players, games, rooms) with server updates
- Handle incoming messages (chat, private messages, broadcasts, keep-alive)
- Send outgoing commands (game creation, player state changes, chat)
- Manage player targeting and ignore lists
- Notify application UI of state changes via callback adapter pattern
- Support multiple concurrent client instances via static registry

## External Dependencies

- **Includes:** `metaserver_messages.h` (message types and `RoomDescription`), `Logging.h` (macros: `logAnomaly()`), `exception`, `vector`, `map`, `memory`, `set`, `stdexcept`
- **Defined elsewhere (forward declared or external):** 
  - `CommunicationsChannel`, `MessageInflater`, `MessageHandler`, `MessageDispatcher` (network pipeline)
  - `Message`, `ChatMessage`, `BroadcastMessage`, `PrivateMessage`, `PlayerListMessage`, `RoomListMessage`, `GameListMessage`, `RemoteHubListMessage`, `SetPlayerDataMessage` (message types)
  - `MetaserverPlayerInfo`, `GameListMessage::GameListEntry` (data types from messages.h)
  - `GameDescription`, `RemoteHubServerDescription` (from messages.h)
  - Standard library types (`uint8`, `uint16`, `uint32`, etc. ΓÇö likely typedefs)

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

# Source_Files/Network/network.h
## File Purpose
Public network subsystem interface for the Aleph One game engine. Exposes APIs for multiplayer game setup (player gathering, joining), in-game synchronization, chat, and network diagnostics. Implements a state machine for network session lifecycle.

## Core Responsibilities
- Define network session states and state transitions
- Player gathering (host) and joining (client) mechanics
- Chat and event callbacks for UI integration
- Game data distribution and synchronization
- Network diagnostics (latency, jitter, error tracking)
- UDP socket management and packet handling
- Capabilities negotiation between clients
- Remote hub (server relay) support for NAT traversal

## External Dependencies

- **cseries.h, cstypes.h:** Basic types (`uint16`, `int16`, `byte`, `OSErr`, `Rect`)
- **CommunicationsChannel.h:** TCP socket abstraction (`TCPsocket`, `IPaddress`, message queuing)
- **network_capabilities.h:** Capability flags for protocol versioning (gameworld, Lua, zipped data, etc.)
- **Pinger.h:** ICMP latency diagnostics (`NetCreatePinger()`, `NetGetPinger()`)
- Standard library: `<string>`, `<vector>`, `<memory>` (for `std::shared_ptr`, `std::weak_ptr`)
- Symbols defined elsewhere: `entry_point`, `player_start_data`, `UDPpacket`, `NetworkInterface`, `TCPlistener`, `MessageInflater`, `MessageHandler`

# Source_Files/Network/network_capabilities.cpp
## File Purpose
Implements static string constant definitions for the `Capabilities` class, which manages version identifiers and feature capability flags for the Aleph One network layer. These constants represent supported network protocols and features that gatherers (servers) and joiners (clients) negotiate during multiplayer game setup.

## Core Responsibilities
- Defines static string constants that identify specific network capability categories
- Provides human-readable capability names for map access in the inherited `std::map<string, uint32>` container
- Acts as the central registry of feature capability identifiers used throughout the network code
- Supports version negotiation between different network client/server implementations

## External Dependencies
- `#include "network_capabilities.h"` ΓÇô declares the `Capabilities` class and version constants
- `#include <string>` ΓÇô indirectly via header for `std::string` type
- `std::string` type from STL

# Source_Files/Network/network_capabilities.h
## File Purpose
Defines a versioning and capability-negotiation system for network protocol compatibility in Aleph One. Maps feature names (strings) to version numbers to enable gatherers (servers) and joiners (clients) to verify mutual protocol support during connection handshakes.

## Core Responsibilities
- Define version constants for network protocols (gameworld, star, Lua, zipped data, network stats, rugby)
- Provide a map-based container (`Capabilities`) for storing and querying capability versions
- Support capability lookup with bounds checking on key names
- Enable cross-version compatibility checking in network initialization

## External Dependencies
- `cseries.h` ΓÇô platform abstraction and type definitions (e.g., `uint32`)
- `<string>` ΓÇô STL `std::string`
- `<map>` ΓÇô STL `std::map` container

# Source_Files/Network/network_dialog_widgets_sdl.cpp
## File Purpose
Implements SDL-based UI widgets for network-related dialogs in Aleph One. Provides specialized widgets for discovering network players during gathering, displaying in-game player rosters, rendering postgame carnage reports with kill/death statistics, and selecting game levels by type.

## Core Responsibilities
- Manage and display lists of prospective joiners discovered during network gathering
- Render player avatars, names, and dynamic status indicators
- Calculate and visualize kill/death/score bar graphs with scaling and collision avoidance
- Support both individual and team-based player grouping layouts
- Handle level/entry-point selection filtered by game type
- Process user interactions (clicks, keyboard navigation) on network UI elements

## External Dependencies
- **sdl_widgets.h:** Base widget classes (w_list, w_select_button, widget, dialog).
- **SSLP_API.h:** Network service discovery definitions (prospective_joiner_info).
- **PlayerImage_sdl.h:** Avatar rendering (PlayerImage class).
- **network_dialogs.h:** Network structures (net_rank, entry_point).
- **screen_drawing.h:** Text and shape rendering (draw_text, text_width, set_drawing_clip_rectangle, SDL_FillRect).
- **sdl_fonts.h:** Font objects and metrics (font_info, get_ascent, get_line_height).
- **TextLayoutHelper.h:** Text collision avoidance.
- **TextStrings.h:** Localized string lookups (TS_GetCString).
- **player.h, HUDRenderer.h, shell.h, interface.h, network.h:** Game world and network APIs (dynamic_world, NetGetPlayerData, get_dialog_player_color, get_net_color).
- **std::vector, std::string, SDL2, cstdio, cstring:** Standard libraries.

# Source_Files/Network/network_dialog_widgets_sdl.h
## File Purpose
Declares custom SDL widget classes for network game dialogs, including player discovery display, game roster visualization, and level selection for networked multiplayer games.

## Core Responsibilities
- **Player discovery & network list**: `w_found_players` manages SSLP callback integration to display available joinable players
- **Game roster & postgame reporting**: `w_players_in_game2` renders current game participants or postgame carnage statistics with bars, icons, and rankings
- **Level selection**: `w_entry_point_selector` filters available map levels by game type compatibility

## External Dependencies
- **sdl_widgets.h**: Base `widget`, `w_list<T>`, `w_select_button` classes
- **SSLP_API.h**: `SSLP_ServiceInstance` (unused here; SSLP integration in implementation)
- **player.h**: `MAXIMUM_PLAYER_NAME_LENGTH` constant; player team/color enums
- **PlayerImage_sdl.h**: `PlayerImage` class for rendering player sprites
- **network_dialogs.h**: `net_rank` struct (postgame rankings), `entry_point` (defined in map.h)
- Standard: `<vector>` (STL containers for player/entry-point lists)

# Source_Files/Network/network_dialogs.cpp
## File Purpose
Implements UI dialogs for network multiplayer functionality in Aleph One. Handles game gathering (server), joining (client), setup configuration, LAN discovery, metaserver integration, and postgame statistics display.

## Core Responsibilities
- Provide dialog UIs for network game setup, gathering, and joining phases
- Implement LAN game discovery using SSLP (Service Location Protocol)
- Manage player chat during pregame and metaserver phases
- Handle metaserver connectivity for Internet game listing and remote hub selection
- Support dedicated remote hub (dedicated server) game hosting
- Display postgame carnage reports with dynamic graph selection
- Progress dialog management for long-running network operations
- Preference binding and data synchronization between UI and game state

## External Dependencies
- **Network:** `network.h`, `network_games.h`, `network_messages.h` ΓÇö game state and network protocols
- **Metaserver:** `metaserver_dialogs.h`, MetaserverClient, GameAvailableMetaserverAnnouncer ΓÇö Internet game listing
- **LAN discovery:** `SSLP_API.h` ΓÇö service location protocol
- **UI/Dialog:** `network_dialog_widgets_sdl.h`, `csdialogs.h` ΓÇö widget and dialog framework
- **Game data:** `map.h`, `game_wad.h`, `shell.h`, `TextStrings.h` ΓÇö level info, strings, startup
- **Preferences:** `preferences.h` ΓÇö player/network/graphics config persistence
- **Sound:** `SoundManager.h` ΓÇö dialog sound effects
- **Progress:** `progress.h` ΓÇö progress dialog support
- **Utilities:** `cseries.h` ΓÇö common C series types and macros

# Source_Files/Network/network_dialogs.h
## File Purpose
Header file declaring network game UI dialogs for Aleph One. Defines classes for hosting/gathering players, joining games, and configuring network game parameters, plus carnage report (postgame statistics) functionality.

## Core Responsibilities
- Abstract dialog classes for network game setup workflow (`GatherDialog`, `JoinDialog`, `SetupNetgameDialog`)
- Dialog control IDs, string resource IDs, and game configuration enums
- Player ranking structure and postgame carnage report declarations
- Factory methods for platform-specific dialog implementations (determined at link-time)
- Chat and network callback interfaces for dialogs

## External Dependencies
- `player.h` ΓÇö `MAXIMUM_NUMBER_OF_PLAYERS`, `player_info`
- `network.h` ΓÇö `game_info`, `player_info`, `prospective_joiner_info`, `GatherCallbacks`, `ChatCallbacks`
- `network_private.h` ΓÇö `JoinerSeekingGathererAnnouncer`
- `FileHandler.h` ΓÇö `FileSpecifier`
- `network_metaserver.h` ΓÇö `MetaserverClient`
- `metaserver_dialogs.h` ΓÇö `GlobalMetaserverChatNotificationAdapter`, `GameAvailableMetaserverAnnouncer`
- `shared_widgets.h` ΓÇö `ButtonWidget`, `EditTextWidget`, `SelectorWidget`, `PlayersInGameWidget`, `EditNumberWidget`, `ToggleWidget`, `EnvSelectWidget`, `ColorfulChatWidget`, etc.
- `preferences_widgets_sdl.h` ΓÇö SDL widget types
- Standard library: `<string>`, `<map>`, `<set>`

# Source_Files/Network/network_dummy.cpp
## File Purpose

Provides stub/dummy implementations of the network subsystem API. Used when building without networking support or as a fallback, allowing the game to run in single-player mode while maintaining the same function signatures as the real network implementation.

## Core Responsibilities

- Stub synchronization functions for network game state
- Return safe single-player defaults for player/game queries
- Prevent network-specific cheats from being enabled
- Provide no-op cleanup and state management
- Maintain API compatibility with the real network module (network.cpp)

## External Dependencies

- **Included headers:**
  - `cseries.h` ΓÇö Base types and platform definitions
  - `map.h` ΓÇö `entry_point` struct definition (for `NetChangeMap` parameter)
  - `network.h` ΓÇö Function declarations / API contract
  - `network_games.h` ΓÇö Game-mode-specific declarations
- **Defined elsewhere:** All structures and game state globals referenced in declarations (e.g., `entry_point`)

# Source_Files/Network/network_games.cpp
## File Purpose
Implements multiplayer game mode logic and scoring for Aleph One's network games. Manages state tracking, ranking calculations, and per-tick game updates for game modes including King of the Hill, Capture the Flag, Tag, Rugby, and Defense.

## Core Responsibilities
- Calculate player and team rankings based on game type and statistics
- Initialize and update game-mode-specific state (beacons, ball carriers, "it" status)
- Determine game-over conditions and format end-game messages
- Compute on-screen compass/beacon directions for gameplay objectives
- Handle per-tick scoring logic (time tracking, goal detection, ball carriers)
- Format ranking displays for HUD and post-game screens

## External Dependencies
- **map.h:** `dynamic_world`, `map_polygons`, `GET_GAME_TYPE()`, `GET_GAME_OPTIONS()`, `get_polygon_data()`, `_polygon_is_hill`, `_polygon_is_base`
- **player.h:** `player_data`, `current_player_index`, `get_player_data()`, `PLAYER_IS_DEAD()`, `PLAYER_IS_TOTALLY_DEAD()`, team color constants
- **items.h:** `find_player_ball_color()`, `BALL_ITEM_BASE`
- **lua_script.h:** `GetLuaScoringMode()`, `GetLuaGameEndCondition()`, Lua compass state arrays
- **game_window.h:** `mark_player_network_stats_as_dirty()`
- **cseries.h:** `temporary` (string buffer), string formatting macros
- **weapons.c** (external): `destroy_players_ball()` (implemented elsewhere per comment)
- **SoundManager.h:** Sound playback (via `play_object_sound()`)

# Source_Files/Network/network_games.h
## File Purpose

Declares the API for managing network multiplayer game state, including initialization, per-frame updates, player/team rankings, scoring, and compass/map UI features for networked matches.

## Core Responsibilities

- Initialize and update network game state and win conditions
- Track and calculate player and team rankings/scores
- Format ranking and scoring text for HUD display
- Determine game-over conditions in networked matches
- Manage network compass state and beacon display for players
- Report kill events (player-on-player damage)
- Query game mode capabilities (scoring, ball physics, etc.)

## External Dependencies

- **Includes:** `player.h` (defines `player_data`, `NUMBER_OF_TEAM_COLORS`)
- **Defined elsewhere:** Player and team data structures, game mode enumeration, damage tracking

# Source_Files/Network/network_messages.cpp
## File Purpose
Implements serialization and deserialization of network protocol messages for Aleph One multiplayer game setup and communication. Handles conversion of message objects to/from byte streams, including compression of large data chunks using zlib.

## Core Responsibilities
- Serialize message objects to byte streams via `reallyDeflateTo()` methods
- Deserialize byte streams back to message objects via `reallyInflateFrom()` methods
- Compress/decompress large message payloads (maps, physics, Lua scripts)
- Serialize/deserialize player network information (addresses, ports, player data)
- Support variable-length network data (capabilities, chat messages, topology info)
- Provide utility functions for string and structured data I/O across stream types

## External Dependencies
- **zlib.h** ΓÇö `compress()`, `uncompress()` for payload compression
- **AStream.h** ΓÇö `AIStream`, `AIStreamBE`, `AOStream`, `AOStreamBE` for typed I/O
- **network_messages.h** ΓÇö Message class definitions and enum constants
- **network_private.h** ΓÇö `NetPlayer`, `NetTopology`, `NetServer`, `ClientChatInfo`, `NetworkStats`, enum tags
- **Logging.h** ΓÇö `logWarning()` macro for error reporting
- **cseries.h** ΓÇö Platform types and utilities

# Source_Files/Network/network_messages.h
## File Purpose
Defines message type constants and classes for network communication in Aleph One's game setup and multiplayer protocol. Implements a type-safe message framework for negotiating game join, capabilities, topology, large data distribution (map/physics/Lua), chat, and remote hub commands over TCPMess.

## Core Responsibilities
- Enumerate message type IDs (kHELLO_MESSAGE through kREMOTE_HUB_REQUEST_MESSAGE)
- Provide template-based message classes for type-safe, compile-time message ID binding
- Define concrete message classes for join handshake, capabilities negotiation, player acceptance
- Support large data distribution with optional compression (zipped variants)
- Implement serialization/deserialization via AStream deflate/inflate pattern
- Manage client connection state and message dispatch handlers

## External Dependencies
- **cseries.h**: Core platform/type definitions (int16, uint32, assert)
- **AStream.h**: AIStream, AOStream classes for endian-aware serialization (operator>>, operator<<)
- **Message.h**: SmallMessageHelper (deflate/inflate), BigChunkOfDataMessage, SimpleMessage<T>, DatalessMessage<tMessageType>, UninflatedMessage
- **network_capabilities.h**: Capabilities class (map of stringΓåÆuint32 feature versions)
- **network_private.h**: NetPlayer, NetTopology, CommunicationsChannel, MessageDispatcher, MessageHandler, ClientChatInfo, RemoteHubCommand enum, prospective_joiner_info struct
- **Standard library**: `<string>`, `<memory>` (std::string, std::shared_ptr, std::unique_ptr, std::vector)
- **SDL2**: Uint8, Uint16 types from SDL.h

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

## External Dependencies
- **cstypes.h**: Sized integer types (int16, uint8, uint32)
- **network.h**: Public network subsystem interface (includes game_info, player_info, network capabilities)
- **SSLP_API.h**: Service location discovery protocol (SSLP_ServiceInstance, SSLP discovery functions)
- **\<memory\>**: C++ standard library (for std::shared_ptr usage in forward-declared types)
- **Forward declarations**: CommunicationsChannel, MessageDispatcher, MessageHandler, Message (defined elsewhere)

# Source_Files/Network/network_star.h
## File Purpose
Header defining the star topology network protocol for Aleph One multiplayer games. Establishes message types, packet structures, and function interfaces for hub (server) and spoke (client) roles communicating via UDP.

## Core Responsibilities
- Define message type constants and packet magic bytes for star topology protocol
- Declare hub initialization, packet handling, and lifecycle management
- Declare spoke (client) initialization, input synchronization, and state queries
- Provide preference configuration interfaces for both hub and spoke modes
- Establish tick-based action queue infrastructure for action flag buffering

## External Dependencies
- `TickBasedCircularQueue.h`: `ConcreteTickBasedCircularQueue<T>`, `WritableTickBasedCircularQueue<T>`
- `ActionQueues.h`: Action flag queue management (included but not directly used in this header)
- `NetworkInterface.h`: `IPaddress`, `UDPpacket`, `UDPsocket`
- `map.h`: `TICKS_PER_SECOND` constant (30 ticks/second)
- `<stdio.h>`: Standard I/O (likely for logging/debugging)
- `InfoTree` class (declared forward; defined elsewhere for preferences)

# Source_Files/Network/network_star_hub.cpp
## File Purpose
Implements the hub (server) side of a star-topology network protocol for synchronized multiplayer games. Receives action flags from players (spokes), maintains tick-based circular queues, calculates per-player timing offsets, synthesizes flags for lagging players (bandwidth reduction), and broadcasts synchronized game state to all connected players.

## Core Responsibilities
- Initialize/cleanup hub background tasks and state; manage graceful shutdown
- Deserialize incoming game data packets, acknowledgments, identification, and ping requests
- Track per-player connectivity, acknowledgments, ack history, address mapping (NAT-friendly)
- Maintain synchronized tick-based action flag queues for all players
- Calculate and adjust per-player timing offsets using windowed Nth-element statistics
- Handle player net-death detection and propagation
- Synthesize action flags for lagging players (bandwidth reduction feature)
- Serialize and broadcast game state packets with acknowledgments, timing messages, and action flags
- Calculate and report per-player latency and jitter statistics

## External Dependencies
- **Includes**: `network_star.h` (public interface), `TickBasedCircularQueue.h`, `AStream.h` (big-endian I/O), `Logging.h`, `WindowedNthElementFinder.h` (Nth-element statistics), `mytm.h` (task/mutex), `crc.h`, `player.h` (action flag masks)
- **External symbols**: `NetDDPSendFrame()`, `NetGetPlayerData()`, `spoke_received_network_packet()`, `myXTMSetup()`, `myTMRemove()`, `myTMCleanup()`, `take_mytm_mutex()`, `release_mytm_mutex()`, `calculate_data_crc_ccitt()`, `local_random()`

# Source_Files/Network/network_star_spoke.cpp
## File Purpose
Implements the client-side ("spoke") component of a star-topology multiplayer network protocol. Handles packet exchange with a central hub server, manages player action flags, detects network death, and adjusts timing for synchronization.

## Core Responsibilities
- Initialize/teardown spoke state with hub address, player queues, and player roster
- Receive and validate game data packets from hub (CRC checking, packet parsing)
- Manage queues of action flags (outgoing, unconfirmed, locally generated) and track acknowledgments
- Enqueue action flags for all players at each tick, generating net-dead flags when necessary
- Measure and apply latency-based timing adjustments to keep client in sync with hub
- Detect hub silence and declare net-dead on timeout
- Send identification packets to hub (NAT-friendly) and periodic data packets with ACKs
- Track display latency for UI feedback
- Parse configuration/preferences (timing windows, net-death thresholds, send periods)

## External Dependencies
- **AStream.h:** `AIStreamBE`, `AOStreamBE` for binary serialization/deserialization (big-endian)
- **mytm.h:** `myXTMSetup()`, `myTMRemove()`, `myTMCleanup()`, mutex operations for timer tasks
- **network_private.h:** `kPROTOCOL_TYPE`, `NET_DEAD_ACTION_FLAG` constant
- **WindowedNthElementFinder.h:** Windowed percentile calculation for latency statistics
- **vbl.h:** `parse_keymap()` to read local player input (defined elsewhere)
- **CircularByteBuffer.h:** Included but not directly used in this file
- **Logging.h:** Logging macros (`logContextNMT`, `logTraceNMT`, `logDumpNMT`, `logWarningNMT`)
- **crc.h:** `calculate_data_crc_ccitt()` for packet validation
- **player.h:** `make_player_really_net_dead()` (external function)
- **InfoTree.h:** Configuration tree for preferences
- **std::map:** Message type dispatch table
- **Defined elsewhere:** `hub_received_network_packet()`, `NetDDPSendFrame()`, `NetGetPinger()`, `TICKS_PER_SECOND`, `UDPpacket` structure

# Source_Files/Network/network_udp.cpp
## File Purpose
Implements UDP socket communication layer for the Aleph One game engine, providing a thin wrapper around UDP sockets with an asynchronous receiving thread. Corresponds functionally to AppleTalk DDP on legacy systems. Handles opening/closing sockets, dispatching received packets via callback, and sending frames to remote machines.

## Core Responsibilities
- Create and manage a UDP socket bound to a configurable port
- Spawn and manage an asynchronous receiving thread that polls for incoming packets
- Dispatch received packets to a registered callback handler (with mutex synchronization)
- Validate and send UDP frames to remote addresses
- Cleanly shutdown socket and thread on close

## External Dependencies
- **SDL2/SDL_thread.h** ΓÇô Thread creation and management
- **thread_priority_sdl.h** ΓÇô `BoostThreadPriority()` for elevating listener thread priority
- **cseries.h** ΓÇô Utility macros (e.g., `fdprintf`)
- **network_private.h** ΓÇô Network constants and type definitions
- **mytm.h** ΓÇô `take_mytm_mutex()`, `release_mytm_mutex()` for synchronization
- **Defined elsewhere:** `NetGetNetworkInterface()`, `UDPsocket` class, `ddpMaxData` constant, `UDPpacket`, `IPaddress`

# Source_Files/Network/NetworkGameProtocol.h
## File Purpose
Abstract interface defining the contract for network game protocol implementations in Aleph One. Handles synchronization, packet distribution, network timing, and client-side action prediction across networked players.

## Core Responsibilities
- Define entry/exit points for network game lifecycle (Enter, Sync, UnSync)
- Manage incoming UDP packet processing
- Provide network timing information
- Track unconfirmed action flags for client-side prediction
- Check for world state updates during gameplay

## External Dependencies
- `network_private.h`: Provides `UDPpacket`, `NetTopology`, `NetPlayer`, game/player info structures
- Concrete implementations expected to handle actual packet format, serialization, and distribution logic

# Source_Files/Network/NetworkInterface.cpp
## File Purpose
Implements a C++ abstraction layer over the ASIO networking library, providing simple wrappers for TCP and UDP socket operations. Part of the Aleph One game engine's network subsystem.

## Core Responsibilities
- Manage IP address objects with ASIO endpoint conversion
- Wrap UDP sockets with methods for unicast, broadcast, and async receive
- Wrap TCP sockets for stream-based communication with non-blocking support
- Implement TCP listener/acceptor for accepting incoming connections
- Provide a central `NetworkInterface` factory that creates and manages all socket types
- Handle DNS resolution of hostnames to IP addresses

## External Dependencies
- **`<asio.hpp>`**: Asio standalone networking library; provides `ip::tcp::socket`, `ip::udp::socket`, `ip::address`, `endpoint`, and `io_context`
- **`<string>`, `<optional>`, `<array>`**: Standard library utilities
- **Defined elsewhere:** `UDPpacket` struct uses `IPaddress` and buffer; all socket classes reference ASIO objects and the shared `io_context`

# Source_Files/Network/NetworkInterface.h
## File Purpose
Provides a C++ abstraction layer over ASIO for network operations in a game engine. Encapsulates UDP and TCP socket management, server listeners, and IP address handling for both client and server networking scenarios.

## Core Responsibilities
- Define `IPaddress` class to wrap IP addresses and ports with conversion utilities
- Implement `UDPsocket` for datagram communication (broadcast, send, receive, async)
- Implement `TCPsocket` for stream-based connections
- Implement `TCPlistener` for server-side connection acceptance
- Provide `NetworkInterface` as the main facade for socket creation and DNS resolution
- Define `UDPpacket` structure for packet data with embedded address information

## External Dependencies
- **`<asio.hpp>`:** Asynchronous I/O library (Boost.ASIO or standalone); provides socket, resolver, and endpoint abstractions
- **`<string>`, `<optional>`, `<array>`:** C++ standard library
- **ASIO namespaces used:** `asio::ip::address`, `asio::ip::udp`, `asio::ip::tcp`, `asio::io_context`

# Source_Files/Network/Pinger.cpp
## File Purpose
Implements network ping functionality to measure latency to registered IP addresses. Sends UDP-based ping requests and collects response times with timeout support. Part of the Aleph One game engine's networking subsystem for connection quality monitoring.

## Core Responsibilities
- Register IPv4 addresses for ping monitoring with unique identifiers
- Send UDP ping request packets with configurable retry counts
- Store ping timing metadata (sent tick, received tick)
- Collect and return response time measurements with timeout
- Thread-safe access via mutex protection during packet transmission

## External Dependencies
- **Pinger.h**: Class declaration, PingAddress struct.
- **network_star.h**: Packet constants (`kPingRequestPacket`, `kStarPacketHeaderSize`), UDPpacket type, `NetDDPSendFrame()`.
- **network_private.h**: Networking infrastructure.
- **crc.h**: `calculate_data_crc_ccitt()` ΓÇô computes 16-bit CCITT CRC.
- **mytm.h**: `machine_tick_count()`, `take_mytm_mutex()`, `release_mytm_mutex()`, `sleep_for_machine_ticks()`.
- **AStream.h**: `AOStreamBE` ΓÇô big-endian output serialization stream.
- **Standard library**: `<unordered_map>`, `<atomic>`, `<algorithm>` (std::max).

# Source_Files/Network/Pinger.h
## File Purpose
Defines the `Pinger` class for managing ICMP ping requests and measuring latency to registered IPv4 addresses. Part of the Aleph One game engine's network layer for health monitoring and connection quality assessment.

## Core Responsibilities
- Register IPv4 addresses for ping monitoring with unique identifiers
- Send ICMP ping requests to registered addresses with configurable retry and filtering
- Track ping transmission and response timestamps to measure round-trip latency
- Store incoming ping responses (called by network receive handler)
- Retrieve response times for all or specific registered addresses with timeout support

## External Dependencies
- `NetworkInterface.h`: Provides `IPaddress` type (wraps asio address/port)
- `<unordered_map>`: Hash map for O(1) response lookup by identifier
- `<atomic>`: `std::atomic_uint32_t` for lock-free thread-safe timestamp updates from async receive thread
- Conditioned on `!DISABLE_NETWORKING` preprocessor guard

# Source_Files/Network/PortForward.cpp
## File Purpose
Implements automatic UPnP port forwarding for network connectivity. Discovers Internet Gateway Devices on the local network, maps a specified port (TCP and UDP) through UPnP to enable external access, and cleans up mappings on shutdown using RAII patterns.

## Core Responsibilities
- Discover UPnP-capable Internet Gateway Devices via `upnpDiscover()`
- Validate and select a valid IGD from discovered devices
- Configure bidirectional TCP and UDP port mappings through the IGD
- Manage lifecycle of port mappings (create on construction, destroy on destruction)
- Handle errors and rollback partial state (e.g., roll back TCP if UDP mapping fails)
- Encapsulate miniupnpc resource management behind unique_ptr deleters

## External Dependencies
- **miniupnpc library:**
  - `<miniupnpc/upnpcommands.h>` ΓÇö UPnP command functions (UPNP_AddPortMapping, UPNP_DeletePortMapping, UPNP_GetValidIGD)
  - `<miniupnpc/miniupnpc.h>` ΓÇö UPnP device discovery (upnpDiscover, UPNPUrls, UPNPDev, IGDdatas, freeUPNPDevlist)
- **Standard library:**
  - `<sstream>` ΓÇö ostringstream for error message formatting
  - `<memory>`, `<stdexcept>` ΓÇö unique_ptr, std::runtime_error (via header)
- **Defined elsewhere (miniupnpc):**
  - `upnpDiscover()`, `UPNP_GetValidIGD()`, `UPNP_AddPortMapping()`, `UPNP_DeletePortMapping()`, `FreeUPNPUrls`, `freeUPNPDevlist`

# Source_Files/Network/PortForward.h
## File Purpose
Provides UPnP port forwarding functionality for both TCP and UDP protocols. Wraps the miniupnpc library with RAII semantics and exception-based error handling to automatically manage port mappings and cleanup.

## Core Responsibilities
- Manage UPnP device discovery and port mapping setup
- Maintain resource lifecycle for UPnP URLs and device lists using smart pointers
- Provide exception-based error signaling for port forwarding operations
- Abstract miniupnpc library details behind a C++ interface
- Support simultaneous TCP and UDP port forwarding for a single port

## External Dependencies
- `miniupnpc/miniupnpc.h` ΓÇö UPnP client library (UPNPUrls, UPNPDev, FreeUPNPUrls, freeUPNPDevlist)
- `<stdexcept>` ΓÇö std::runtime_error base class
- `<memory>` ΓÇö std::unique_ptr
- `cseries.h` ΓÇö project common definitions
- Conditional compile: `HAVE_MINIUPNPC` preprocessor guard

# Source_Files/Network/SSLP_API.h
## File Purpose
Public API header for SSLP (Simple Service Location Protocol), a custom service discovery protocol for locating and advertising networked services. Designed for the Aleph One project to enable cross-network player discovery, replacing AppleTalk-based service location.

## Core Responsibilities
- Expose service discovery API for client-side service location
- Expose service advertisement API for server-side service exposure
- Define callback mechanism for service lifecycle events (found, lost, renamed)
- Coordinate periodic processing of discovery and advertisement logic
- Provide service instance representation with type, name, and network address

## External Dependencies
- **NetworkInterface.h**: Provides `IPaddress` type (encapsulates host and port in network byte order)
- **Implicit C runtime**: `NULL` pointer constant

# Source_Files/Network/SSLP_limited.cpp
## File Purpose
Implements the Simple Service Location Protocol (SSLP) for discovering and advertising network services in Aleph One. Uses a non-threaded, pump-based design where the application's main thread periodically calls `SSLP_Pump()` to perform discovery and advertisement work. Designed to be lightweight, with automatic timeout and garbage collection of stale service entries.

## Core Responsibilities
- **Service Discovery**: Locate remote services by broadcasting FIND packets and processing HAVE responses
- **Service Advertisement**: Respond to discovery requests and allow local services to be found by peers
- **Packet Processing**: Receive, parse, and interpret SSLP protocol messages (FIND, HAVE, LOST)
- **Instance Tracking**: Maintain linked list of discovered service instances with timestamp-based timeout
- **Callback Notification**: Invoke registered callbacks when services are found, lost, or renamed
- **Resource Management**: Initialize/shutdown UDP socket and clean up found instances

## External Dependencies
- **SDL2:** `SDL_SwapBE32()` for endian conversion, `SDL_thread.h` (included but not used in limited implementation)
- **Network Interface:** `NetGetNetworkInterface()`, `UDPsocket` abstraction, `UDPpacket`, `IPaddress` with `.address()` and `.port()` methods
- **Logging:** `logTrace()`, `logContext()`, `logNote()` macros from `Logging.h`
- **Time:** `machine_tick_count()` from `csmisc.h` (millisecond precision)
- **C Standard:** `memcpy()`, `strncpy()`, `strncmp()`, `malloc()`, `free()`

# Source_Files/Network/SSLP_Protocol.h
## File Purpose
Defines the Simple Service Location Protocol (SSLP) specificationΓÇöa lightweight network service discovery protocol for the Aleph One game engine. Specifies packet structure, message types, and protocol semantics for discovering network game instances and players across non-AppleTalk networks.

## Core Responsibilities
- Define the SSLP packet format and wire protocol specification
- Document message types: FIND (service discovery request), HAVE (service advertisement), LOST (service retirement)
- Specify magic numbers, version, and port constants for protocol identification
- Establish naming constraints for service types and instance names
- Document protocol behavior, delivery guarantees, and usage patterns

## External Dependencies
- Standard C types: `uint32_t`, `uint16_t`, `char` (from `<stdint.h>`)
- No external symbols or library dependencies; pure protocol specification

**Notes**: Service type and name fields are treated as C-strings for comparison (match on first `\0`); NULL termination is not required if all `SSLP_MAX_*_LENGTH` bytes are used. In Aleph One, service types are player identifiers and service names are player join handles.

# Source_Files/Network/StandaloneHub/standalone_hub_main.cpp
## File Purpose
Main entry point for the Aleph One standalone network hub server. Orchestrates multiplayer game initialization, state transitions, and real-time message processing. Manages the hub's three operational states: waiting for gatherer, active game, and shutdown.

## Core Responsibilities
- Initialize hub infrastructure (logging, preferences, networking, keyboard input)
- Manage game state machine with three states (waiting ΓåÆ in_progress ΓåÆ quit)
- Load and decompress game map data from WAD format
- Process network messages during active gameplay
- Synchronize gatherer connection and map transitions
- Handle graceful error conditions and shutdown

## External Dependencies
- **Networking:** `network_star.h` (NetProcessMessagesInGame, NetUnSync, NetChangeMap, NetSync, NetStart, hub_is_active)
- **Hub Management:** `StandaloneHub.h` (singleton instance, game data, gatherer communication)
- **Game State:** `map.h` (dynamic_world, map initialization), `wad.h`, `game_wad.h` (WAD decompression)
- **System Init:** `preferences.h` (network_preferences, DefaultHubPreferences), `mytm.h`, `vbl.h`, `Logging.h`

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

## External Dependencies

- **Includes:** `StandaloneHub.h`, `MessageInflater.h`, `network_messages.h`
- **Network layer (defined elsewhere):** `NetEnter`, `NetExit`, `NetSetDefaultInflater`, `NetProcessNewJoiner`, `NetGather`, `NetCancelGather`, `NetCheckForNewJoiner`, `NetSetCapabilities`, `sleep_for_machine_ticks`, `machine_tick_count`
- **Messaging (defined elsewhere):** `CommunicationsChannelFactory`, `CommunicationsChannel`, `Message` subclasses, `Capabilities`
- **Scripting (defined elsewhere):** `DeferredScriptSend`

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

## External Dependencies
- `MessageInflater.h` ΓÇô message deserialization/inflation machinery
- `network_messages.h` ΓÇô concrete Message subclasses (TopologyMessage, MapMessage, PhysicsMessage, CapabilitiesMessage, etc.)
- `CommunicationsChannelFactory`, `CommunicationsChannel` ΓÇô TCP/network layer (defined elsewhere)
- `Message`, `Capabilities`, `NetTopology` ΓÇô base types (defined elsewhere, likely in network_private.h or network_capabilities.h)

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

## External Dependencies
- **cseries.h**: Core platform/endianness definitions, data types.
- **StarGameProtocol.h**: Class declaration.
- **network_star.h**: Hub/spoke low-level functions (`hub_initialize`, `spoke_initialize`, etc.), message type constants, `TickBasedActionQueue` typedef.
- **TickBasedCircularQueue.h**: `WritableTickBasedCircularQueue`, `ConcreteTickBasedCircularQueue` templates.
- **player.h**: `GetRealActionQueues()` (defined elsewhere); player constants/macros.
- **interface.h**: `process_action_flags()` (defined in vbl.c).
- **InfoTree.h**: XML/INI tree configuration.
- **Defined elsewhere:** `hub_initialize`, `hub_cleanup`, `hub_received_network_packet`, `spoke_initialize`, `spoke_cleanup`, `spoke_received_network_packet`, `spoke_get_net_time`, `spoke_get_unconfirmed_flags_queue`, `spoke_get_smallest_unconfirmed_tick`, `spoke_check_world_update`, `HubParsePreferencesTree`, `SpokeParsePreferencesTree`, `DefaultHubPreferences`, `DefaultSpokePreferences`, `HubPreferencesTree`, `SpokePreferencesTree`.

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

## External Dependencies
- **Base class**: `NetworkGameProtocol` (defines abstract interface)
- **Network types**: `UDPpacket`, `NetTopology`, `int32` (from `network_private.h` via NetworkGameProtocol.h)
- **Configuration**: `InfoTree` class (forward declared; defined elsewhere)
- **Standard**: `<stdio.h>` (likely for logging)

# Source_Files/Network/Update.cpp
## File Purpose
Implements the Update class, which checks for new software versions by fetching update metadata over HTTP in a background thread. It compares the remote version with the local version and reports whether an update is available.

## Core Responsibilities
- Manage initialization and lifecycle of the update-check background thread
- Perform HTTP GET requests to a remote update server
- Parse version metadata (date version and display version) from HTTP responses
- Compare semantic versions to detect available updates
- Provide thread-safe access to update status and version information via singleton pattern

## External Dependencies
- **HTTPClient** (`HTTP.h`) ΓÇô Makes GET requests and buffers the response.
- **SDL2/SDL_thread.h** ΓÇô Thread creation and synchronization (`SDL_CreateThread`, `SDL_WaitThread`).
- **boost::tokenizer, boost::algorithm::string::predicate** ΓÇô String parsing (tokenization by newlines, prefix matching).
- **alephversion.h** ΓÇô Provides `A1_UPDATE_URL` and `A1_DATE_VERSION` constants.

# Source_Files/Network/Update.h
## File Purpose
Defines the `Update` singleton class that manages background checking for software updates. Uses SDL threading to perform non-blocking update checks and exposes the current status and available version information to the rest of the application.

## Core Responsibilities
- Implement singleton pattern for centralized update management
- Check for new versions asynchronously in a background thread
- Track update check status (checking, failed, available, unavailable)
- Store and expose new version display string when an update is detected
- Manage SDL thread lifecycle for background operations

## External Dependencies
- `<string>` ΓÇö Standard library strings for version storage
- `<SDL2/SDL_thread.h>` ΓÇö Cross-platform threading primitives
- `cseries.h` ΓÇö Aleph One utilities (likely macros, type defs, `assert`)
- Network code (not visible; likely internal HTTP library) ΓÇö called by `Thread()` implementation


