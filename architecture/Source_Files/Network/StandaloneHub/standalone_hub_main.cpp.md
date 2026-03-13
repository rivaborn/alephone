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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `StandaloneHubState` | enum | Hub operational states: _waiting_for_gatherer, _game_in_progress, _quit |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `log_dir` | DirectorySpecifier | extern | Logging output directory path |

## Key Functions / Methods

### initialize_hub
- **Signature:** `static void initialize_hub(short port)`
- **Purpose:** One-time initialization of all hub subsystems
- **Inputs:** port ΓÇô network port for game traffic
- **Side effects:** Creates log directory; initializes global string sets, preferences, keyboard, marathon engine
- **Calls:** `InitDefaultStringSets()`, `get_data_path()`, `DefaultHubPreferences()`, `mytm_initialize()`, `initialize_keyboard_controller()`, `initialize_marathon()`
- **Notes:** Must be called exactly once before main loop; allocates persistent global state

### hub_init_game
- **Signature:** `static bool hub_init_game(void)`
- **Purpose:** Initialize game state and load map from gatherer
- **Side effects:** Decompresses WAD buffer; modifies `dynamic_world`
- **Calls:** `StandaloneHub::Instance()->GetMapData()`, `inflate_flat_data()`, `get_dynamic_data_from_wad()`, `free_wad()`
- **Notes:** Returns false if map unavailable; cleans up allocations on failure

### hub_game_in_progress
- **Signature:** `static bool hub_game_in_progress(bool& game_is_done)`
- **Purpose:** Main game processing loop ΓÇô handles network I/O and level transitions
- **Outputs/Return:** bool (false on fatal error); sets game_is_done flag
- **Side effects:** Processes network messages; may transition to next level or end game
- **Calls:** `NetProcessMessagesInGame()`, `NetUnSync()`, `NetChangeMap()`, `NetSync()`

### hub_host_game
- **Signature:** `static bool hub_host_game(bool& game_has_started)`
- **Purpose:** Establish gatherer connection and initialize first game
- **Calls:** `StandaloneHub::Init()`, `WaitForGatherer()`, `GetGameDataFromGatherer()`, `hub_init_game()`, `SetupGathererGame()`, `NetStart()`
- **Notes:** May return true without starting if waiting for players; game starts only if gathering_done is true

### main_loop_hub
- **Signature:** `static void main_loop_hub(void)`
- **Purpose:** State machine loop managing hub lifecycle
- **Side effects:** Infinite loop; calls `sleep_for_machine_ticks(1)` between iterations
- **Control flow:** Transitions based on success/failure flags from state handlers

### parse_port
- **Signature:** `static uint16_t parse_port(char* port_arg)`
- **Purpose:** Parse and validate command-line port argument
- **Notes:** Validates string length Γëñ5, numeric only; rejects values > UINT16_MAX

## Control Flow Notes
**State machine** with three states:
1. **_waiting_for_gatherer:** Initialization phase ΓÇô awaits gatherer, receives game data, starts game.
2. **_game_in_progress:** Active gameplay ΓÇô processes messages, handles level transitions, detects game end.
3. **_quit:** Terminal state ΓÇô exits main loop.

Entry: `main()` parses port, calls `initialize_hub()`, then `main_loop_hub()`. Exception handlers catch and log all unhandled exceptions.

## External Dependencies
- **Networking:** `network_star.h` (NetProcessMessagesInGame, NetUnSync, NetChangeMap, NetSync, NetStart, hub_is_active)
- **Hub Management:** `StandaloneHub.h` (singleton instance, game data, gatherer communication)
- **Game State:** `map.h` (dynamic_world, map initialization), `wad.h`, `game_wad.h` (WAD decompression)
- **System Init:** `preferences.h` (network_preferences, DefaultHubPreferences), `mytm.h`, `vbl.h`, `Logging.h`
