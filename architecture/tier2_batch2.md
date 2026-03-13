# Architecture Overview

## Repository Shape

Aleph One organizes as a monolithic C++ engine with core subsystems and supporting utilities. Based on the subsystem overviews provided, the layout includes:

```
alephone/
Γö£ΓöÇΓöÇ Source/
Γöé   Γö£ΓöÇΓöÇ [Core subsystems]            # GameWorld, RenderMain/RenderOther, Sound, Network, CSeries, Files, Input, Lua, etc.
Γöé   Γö£ΓöÇΓöÇ TCPMess/                     # TCP message-based networking (CommunicationsChannel, Message, MessageDispatcher, MessageInflater)
Γöé   Γö£ΓöÇΓöÇ Videos/                      # Recording and playback (Movie, pl_mpeg)
Γöé   Γö£ΓöÇΓöÇ XML/                         # MML parsing and config (InfoTree, XML_MakeRoot, Plugins, XML_LevelScript, QuickSave)
Γöé   Γö£ΓöÇΓöÇ [Interface/Shell/Screen]     # Application lifecycle, rendering, display management
Γöé   ΓööΓöÇΓöÇ [Utilities]                  # Steam shim, resource_manager, wad, FileHandler
Γö£ΓöÇΓöÇ Source/Steam/                    # Steam API parent shim (steamshim_parent.cpp)
Γö£ΓöÇΓöÇ tests/                           # Catch2 test suite
Γöé   Γö£ΓöÇΓöÇ main.cpp                     # Test runner
Γöé   ΓööΓöÇΓöÇ replay_film_test.cpp         # Replay determinism validation
ΓööΓöÇΓöÇ tools/                           # Offline inspection utilities
    Γö£ΓöÇΓöÇ dumprsrcmap.cpp              # Macintosh resource map inspection
    ΓööΓöÇΓöÇ dumpwad.cpp                  # Marathon WAD file inspection
```

---

## Major Subsystems

### TCPMess

**Purpose:**  
Reliable, message-based TCP networking for peer-to-peer multiplayer communication. Manages bidirectional socket connections, asynchronous/synchronous message queueing and dispatch, serialization/deserialization with optional compression, and handler-based message routing by type.

**Key directories / files:**
- `CommunicationsChannel.h/cpp` ΓÇö TCP connection lifecycle, message queue management, async pump and blocking receive APIs
- `Message.h/cpp` ΓÇö Abstract message interface; serialization/deserialization bridges `AIStreamBE`/`AOStreamBE` and raw buffer formats
- `MessageHandler.h` ΓÇö Abstract handler interface with function pointer and member method templates
- `MessageDispatcher.h` ΓÇö Type-ID message router with fallback support
- `MessageInflater.h/cpp` ΓÇö Prototype-pattern registry for message deserialization and instantiation

**Key responsibilities:**
- Establish, maintain, and tear down TCP socket connections
- Queue and transmit outgoing messages with 8-byte header + body framing; batch-flush multiple channels
- Receive, buffer, and parse incoming messages with header validation and timeout enforcement
- Serialize/deserialize messages with optional inflation (decompression)
- Route messages to registered handlers based on message type ID
- Provide synchronous blocking receive with overall and inactivity timeout tracking
- Support message type filtering, cloning, and prototype-based instantiation

**Key dependencies (other subsystems):**
- `NetworkInterface` ΓÇö Socket primitives (`TCPsocket`, `TCPlistener`, `IPaddress`)
- `AStream` (`AIStreamBE`, `AOStreamBE`) ΓÇö Stream-based message serialization
- `CSeries` ΓÇö Timing utilities (`machine_tick_count()`, `sleep_for_machine_ticks()`)

---

### Videos

**Purpose:**  
Gameplay recording and video playback. Encodes game footage and audio into WebM files via background thread processing. Decodes MPEG-1 video and MP2 audio for in-game video playback, supporting both hardware-accelerated (OpenGL) and software rendering paths.

**Key directories / files:**
- `Movie.h/cpp` ΓÇö Singleton interface for recording state, frame queueing, background thread coordination, VP8/Vorbis encoding to WebM
- `pl_mpeg.h` ΓÇö Self-contained MPEG-1 decoder with Program Stream demuxing, frame-accurate seeking, MP2 audio support

**Key responsibilities:**
- Capture video frames from OpenGL framebuffer or SDL software surfaces
- Encode video frames to VP8 codec with configurable bitrate and quality scaling by resolution
- Encode audio to Vorbis codec with OpenALManager loopback capture
- Multiplex audio and video into WebM/Matroska container with frame synchronization
- Manage thread-safe producerΓÇöconsumer pipeline (main thread captures, encode thread processes asynchronously)
- Decode MPEG-1 video frames and MP2 audio with PTS timestamp handling
- Convert between color spaces (ARGBΓåÆI420 YUV, YCrCbΓåÆRGB) for encoding and display
- Support frame-accurate seeking, frame-skip modes, and audio lead-time management for A/V sync

**Key dependencies (other subsystems):**
- `RenderMain`/`RenderOther` ΓÇö Rendered frame buffers (OpenGL framebuffer objects or SDL surfaces)
- `Screen` ΓÇö Viewport information, surface access, OpenGL detection state
- `Sound` (OpenALManager) ΓÇö Audio device loopback capture for recording
- `interface.h` ΓÇö FPS target, input state, user alerts, file dialogs
- External: libvpx, libvorbis, libmatroska, libyuv, SDL2, OpenGL

---

### XML

**Purpose:**  
Centralized configuration, metadata, and scripting management via Marathon Markup Language (MML) XML files. Parses MML to populate game subsystems, orchestrates level-specific scripts (MML, Lua, movies, music), manages plugins with resource patch loading, and handles quick save file metadata.

**Key directories / files:**
- `InfoTree.cpp/h` ΓÇö Type-safe Boost.PropertyTree wrapper; XML/INI parsing with game-type serialization
- `XML_MakeRoot.cpp` ΓÇö Root MML dispatcher; routes subsystem elements to ~60 `parse_mml_*()` handlers
- `XML_ParseTreeRoot.h` ΓÇö Public MML API (`ParseMMLFromFile`, `ParseMMLFromData`, `ResetAllMMLValues`)
- `Plugins.cpp/h` ΓÇö Plugin discovery, validation, resource loading (MML, shapes, sounds, maps), exclusivity enforcement
- `XML_LevelScript.cpp/h` ΓÇö Per-level script orchestration; executes MML/Lua, manages movies, music, load screens
- `QuickSave.cpp/h` ΓÇö Quick save metadata, preview image caching, UI dialogs, LRU cache

**Key responsibilities:**
- Parse XML/INI configuration files via Boost.PropertyTree with bounds checking and enum validation
- Load and reset MML configurations during engine initialization and level transitions
- Route parsed XML elements to subsystem-specific parsers (~60 handlers for weapons, monsters, items, physics, interface, etc.)
- Discover and validate plugins from directories and Steam workshop
- Load plugin resources and enforce mutual exclusivity (single HUD Lua, theme, stats Lua)
- Execute level-specific scripts, manage movie playback, coordinate music playlists, configure load screens
- Manage quick save files with metadata (level name, playtime, player count) and preview images
- Serialize/deserialize game-specific numeric types (fixed-point angles, world units, colors) between XML and memory

**Key dependencies (other subsystems):**
- `FileHandler` ΓÇö Cross-platform file I/O (`FileSpecifier`, `DirectorySpecifier`, `OpenedFile`)
- `GameWorld types` ΓÇö `shape_descriptor`, `damage_definition`, `angle`, `_fixed`, `RGBColor`, `world_point2d/3d`
- ~60 subsystem-specific handlers (`parse_mml_*()`, `reset_mml_*()`) for weapons, monsters, items, interface, physics, etc.
- `Music`, `Lua` subsystems ΓÇö Level script execution
- `Scenario` ΓÇö Active map metadata and resource context
- `Preferences` ΓÇö Solo Lua, stats, network mode flags
- External: Boost.PropertyTree, Boost.Iostreams, Steam workshop API

---

### Steam

**Purpose:**  
Steam API parent shim process. Acts as a bridge between the Steamworks SDK and the game child process. Initializes the Steam SDK, spawns and manages the game as a child process, and relays Steam API calls, callbacks, and workshop operations through bidirectional pipes using a binary protocol.

**Key directories / files:**
- `Source/Steam/steamshim_parent.cpp` ΓÇö Parent process lifecycle, Steam SDK initialization, child process spawning, binary protocol command parsing, async callback relay

**Key responsibilities:**
- Spawn child game process with platform-specific mechanisms (Windows `CreateProcessW` / Unix `fork`+`execvp`)
- Establish bidirectional pipe communication with game child
- Initialize and maintain Steamworks SDK interface pointers and state
- Parse and dispatch binary protocol commands from child process to appropriate Steam API methods
- Handle Steamworks async callbacks and relay results back to child
- Manage workshop item lifecycle operations (upload, update, query, delete)
- Persist achievement and statistics data through Steam UGC/UserStats APIs
- Monitor game overlay activation state
- Set up environment variables for child process Steam runtime integration

**Key dependencies (other subsystems):**
- `shell` (child process) ΓÇö Command-line parsing, initialization, main event loop
- Steamworks SDK ΓÇö Steam API callbacks, async results, workshop metadata, user statistics
- Platform I/O ΓÇö Windows pipes (`CreatePipe`, `WriteFile`, `ReadFile`) / POSIX pipes (`pipe`, `fork`, `execvp`)
- Boost.Filesystem ΓÇö Program binary location, workshop directory path resolution

---

### tests

**Purpose:**  
Testing infrastructure for Aleph One via Catch2 test framework. Validates replay film loading, playback, and random seed consistency to ensure replay determinism across engine updates.

**Key directories / files:**
- `tests/main.cpp` ΓÇö Catch2 test runner entry point; parses and filters engine shell options before framework initialization
- `tests/replay_film_test.cpp` ΓÇö Replay film validation suite; loads, executes, and verifies replay seeding and determinism

**Key responsibilities:**
- Parse and filter engine-specific shell options from command-line arguments before Catch2 initialization
- Load replay files from configurable directory trees and validate format integrity
- Execute replays at maximum speed and verify random seed consistency against filename expectations
- Extract and validate random number state from replay execution
- Provide auto-seeding utility to rename and seed malformed replay files
- Manage test-wide application lifecycle (initialize/shutdown)

**Key dependencies (other subsystems):**
- `shell` ΓÇö Application lifecycle (`initialize_application()`, `shutdown_application()`, `main_event_loop()`, `handle_open_document()`)
- `GameWorld` ΓÇö Random number state (`set_random_seed()`, `get_random_seed()`, `global_random()`)
- `FileHandler` ΓÇö Cross-platform file I/O (`FileSpecifier`, `dir_entry`)
- `shell_options` ΓÇö Configuration struct with `replay_directory` field
- `interface` ΓÇö Replay control (`set_replay_speed()`)
- `preferences` ΓÇö Graphics preferences reference

---

### tools

**Purpose:**  
Command-line utilities for offline inspection and debugging of Marathon/Aleph One data files and binary resources. Standalone programs that analyze WAD files and Macintosh resource maps, extracting and displaying file structure metadata in human-readable format.

**Key directories / files:**
- `tools/dumprsrcmap.cpp` ΓÇö Utility to parse and display Macintosh resource map metadata (types, IDs, sizes)
- `tools/dumpwad.cpp` ΓÇö Utility to parse and display Marathon WAD file structure, headers, directories, and level metadata

**Key responsibilities:**
- Parse command-line arguments and validate file input paths
- Open and read Marathon data files (resource forks; WAD files)
- Parse binary file structures (resource maps, WAD headers, directory entries, level metadata)
- Extract and interpret resource/level metadata (types, IDs, sizes, mission/environment flags, entry points)
- Enumerate data elements (resource types/IDs; WAD tags/entries)
- Decode and display human-readable flag interpretations
- Output inspection results to stdout in formatted text

**Key dependencies (other subsystems):**
- `resource_manager` ΓÇö `open_res_file()`, `get_resource_id_list()`, `get_resource()`, `count_resources()`
- `wad` ΓÇö `open_wad_file_for_reading()`, `read_wad_header()`, `read_directory_data()`
- `FileHandler_SDL` ΓÇö File I/O abstraction (`OpenedFile`, `FileSpecifier`)
- `Packing` ΓÇö Serialization utilities (`StreamToValue()`, `StreamToBytes()`)
- External: SDL2 (`SDL_RWops`)

---

## Key Runtime Flows

### Initialization

1. **Steam parent process** (if Steam build):
   - Call `SteamAPI_InitEx()` to initialize Steamworks SDK
   - Set up environment variables for child process
   - Spawn child game process via `CreateProcessW` (Windows) or `fork`+`execvp` (POSIX)
   - Establish bidirectional pipes for Steam API relay

2. **Game child process** (or standalone):
   - `shell::initialize_application()` parses command-line arguments and shell options
   - **tests subsystem** (if test mode): Filters engine-specific arguments from `argv` before Catch2 initialization
   - Core engine subsystems initialize (GameWorld, RenderMain/RenderOther, Sound, Network, CSeries, Files, Input, Lua)
   - **XML subsystem** invokes `ParseMMLFromFile()` to load default MML configurations:
     - Routes parsed elements to ~60 subsystem-specific `parse_mml_*()` handlers
     - Populates weapons, monsters, items, physics, interface, shapes, damage definitions, etc.
   - **Plugins subsystem** discovers and validates plugins from directories and Steam workshop:
     - Loads plugin MML, resource patches (shapes, sounds, maps)
     - Enforces mutual exclusivity (single HUD Lua, theme, stats Lua)
   - **Videos subsystem** initializes OpenGL detection and color space conversion tables

3. **Per-level initialization**:
   - **XML_LevelScript** loads and executes level-specific scripts from map resource 128:
     - Parses level MML, executes Lua scripts (level-specific)
     - Configures movies, music playlists, load screens
   - **TCPMess** registers message prototypes via `MessageInflater` for netgame
   - **Network** creates `CommunicationsChannel` instances or accepts peer connections

### Per-frame / Main Loop

1. **shell::main_event_loop()** processes user input and advances time
   - Captures input (keyboard, mouse, gamepad) via `Input` subsystem
   - Calls frame update callbacks for all active subsystems

2. **GameWorld** updates (entity logic, physics, AI, weapon firing, projectiles, pathfinding, item collection)

3. **Network** (if netgame active):
   - **TCPMess** calls `pump()` on all active `CommunicationsChannel` instances:
     - Processes queued incoming/outgoing messages asynchronously
     - Routes dispatched messages to registered handlers via `MessageDispatcher`
   - Synchronizes game state across peers

4. **Rendering**:
   - **RenderMain** (software) or **RenderOther** (OpenGL) renders current frame based on player viewpoint
   - **Videos subsystem** (if recording active):
     - Main thread captures rendered frame (OpenGL FBO or SDL surface) and queues to producer buffer
     - Encode thread dequeues asynchronously, encodes to VP8, writes WebM cluster
     - Audio captured from OpenAL loopback, encoded to Vorbis, multiplexed with video

5. **Sound** (OpenAL/SDL_mixer/OPL FM):
   - Updates positional 3D audio based on entity positions
   - Plays sound effects and music

6. **Lua scripting** (HUD, solo, stats):
   - HUD Lua executes to render overlays
   - Solo Lua executes (if single-player level defines script)
   - Stats Lua executes (if enabled)

7. **Steam parent** (if Steam build):
   - Call `SteamAPI_RunCallbacks()` to process async Steam events
   - Parse binary protocol commands from child process pipe
   - Dispatch relay responses back to child

### Shutdown

1. **shell::shutdown_application()**:
   - **Videos subsystem**: Flush remaining video/audio frames to WebM file; stop encode thread
   - **TCPMess**: Flush pending messages on all active channels; close TCP sockets; deallocate queues
   - **Network**: Disconnect from all peers
   - **Music**: Stop playback
   - **Lua**: Deallocate Lua interpreter and HUD scripts
   - **XML**: (No explicit unload; data persists until reset via `ResetAllMMLValues()`)
   - **Plugins**: Unload plugin resources
   - Core engine subsystems clean up (GameWorld, RenderMain/RenderOther, Sound, Input, Files, CSeries)

2. **Steam parent** (if Steam build):
   - Wait for child process exit
   - Close bidirectional pipes
   - Call `SteamAPI_Shutdown()`

---

## Data & Control Boundaries

### Message Ownership & Lifetimes

- **TCPMess**: `CommunicationsChannel` owns queued `Message` objects; dequeued messages are owned by `MessageDispatcher` and handler callbacks. `UninflatedMessage` (wire representation) is deallocated after deserialization. `BigChunkOfDataMessage` manages heap-allocated buffers; caller is responsible for cleanup after message handling.
- **Videos**: `Movie` singleton owns frame/audio buffers in producerΓÇöconsumer ring queue; encode thread dequeues and processes asynchronously. WebM file handle is owned by encode thread until closure.

### Global State & Singletons

- **Movie** (Videos) ΓÇö Singleton managing recording state and frame queue
- **Plugins::instance()** (XML) ΓÇö Singleton for plugin discovery, validation, resource loading
- **QuickSaves** (XML) ΓÇö Singleton for enumerating and managing saved games
- **MessageInflater** (TCPMess) ΓÇö Global prototype registry for message deserialization
- **Steam parent** ΓÇö Maintains single Steamworks SDK interface pointer shared across callback handlers

### Resource Ownership & Cleanup

- **XML**: Owned resources (MML-loaded subsystem state) persist until `ResetAllMMLValues()` or engine shutdown. Plugin resources are loaded/unloaded atomically with exclusivity enforcement.
- **QuickSave**: Preview images cached in LRU memory; evicted on capacity limits or on-demand reload from disk.
- **Files/WAD I/O**: File handles managed by `FileHandler` abstraction; resource lifetime bounded by `OpenedFile` scope.

### Configuration & Reset

- **XML/MML**: `ParseMMLFromFile()` populates subsystem state; `ResetAllMMLValues()` resets all subsystems to defaults. Level-specific scripts override defaults via level resource 128. Pseudo-levels (Default, Restore, End-of-game) apply common MML operations across all maps.

### Process Boundary (Steam)

- **ParentΓÇöchild pipe communication**: Binary protocol encodes Steam API commands and results; messages are stateless (no transaction IDs required). Callback results are queued and flushed to child on polling intervals.

---

## Notable Risks / Hotspots

- **TCPMess message header framing**: 8-byte header format not explicitly specified in overviews; version mismatch or corruption could cause message deserialization failure or infinite packet reads. Timeout enforcement mitigates blocking indefinitely on malformed messages.
- **Videos producerΓÇöconsumer synchronization**: Semaphore-based coordination may deadlock if frame queue fills faster than encode thread processes. No backpressure mechanism documented; main thread may stall if producer buffer is exhausted.
- **XML plugin exclusivity**: Only one HUD Lua, theme, or stats Lua allowed active. Validation occurs at load time; no runtime enforcement prevents plugin conflicts if plugin state is modified mid-game.
- **Replay determinism (tests)**: Random seed consistency is validated against filename expectations; filename encoding format must be stable across versions. Seed extraction from gameplay assumes reproducible RNG state; any change to random number algorithm or subsystem initialization order breaks replay compatibility.
- **Steam parentΓÇöchild process**: Pipe communication is synchronous and blocking on message reads; long-running Steam API callbacks in parent may stall child process main loop. No documented timeout mechanism for stuck parent process.
- **QuickSave LRU cache**: Preview image eviction on capacity limits may cause visible stutter if cache misses trigger disk I/O during gameplay. No prefetch or asynchronous loading strategy documented.
- **Plugins from Steam workshop**: Validation occurs at discovery time; no runtime protection against malformed plugin MML or resource corruption. Resource conflicts resolved via checksums; hash collisions not addressed.
