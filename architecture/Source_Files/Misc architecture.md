# Subsystem Overview

## Purpose

Provides foundational utilities, configuration, and infrastructure services to the Marathon: Aleph One game engine. This subsystem aggregates preferences/dialogs, logging, console commands, replay recording/playback, achievements, input action queueing, statistics collection, and embedded resources (fonts, images). It acts as a bridge between the core game systems and platform-specific implementations.

## Key Files

| File | Role |
|------|------|
| interface.cpp/h | Game state machine; transitions between intro, menus, gameplay, replays, epilogue |
| preferences.cpp/h | Complete preferences/configuration system; all subsystem settings (graphics, network, player, audio, input, environment) |
| vbl.cpp/h | Input polling, action flag capture, replay recording/playback, VBL frame timing (30 Hz timer task) |
| Logging.cpp/h | Thread-safe logging with context stacks, severity levels, file/stderr output |
| Console.cpp/h | Interactive console; text input, command execution, history, macros, kill-message reporting |
| achievements.cpp/h | Achievement validation, Lua trigger generation, Steam integration via `HAVE_STEAM` |
| ActionQueues.cpp/h | Multi-player circular-buffer action flag queues; enqueue/dequeue/peek with zombie state |
| Statistics.cpp/h | Gameplay stats collection from Lua; asynchronous HTTP POST upload to stats server |
| preference_dialogs.cpp/h | OpenGL graphics preferences dialog UI with bindable adapters |
| preferences_widgets_sdl.cpp/h | File/directory selection, crosshair preview, plugin list widgets |
| shared_widgets.cpp/h | Chat history observer pattern; preference binders (string, bool, int, file path) |
| PlayerImage_sdl.cpp/h | SDL-based player sprite rendering; legs/torso composition with appearance state |
| ScenarioChooser.cpp/h | Fullscreen scenario grid selector with keyboard/mouse/controller navigation |
| binders.h | State synchronization protocol (`Bindable<T>`, `ABinder`, `BinderSet`); bidirectional binding |
| CircularQueue.h | Template circular FIFO with O(1) enqueue/dequeue; basis for other queues |
| CircularByteBuffer.cpp/h | Byte circular queue specialization; copy and zero-copy bulk operations with wraparound |
| Random.h | PRNG struct (Marsaglia algorithms: KISS, MWC, LFIB4); independent instances per subsystem |
| VecOps.h | Template vector operations (copy, add, subtract, scale, dot, cross product) |
| WindowedNthElementFinder.h | Sliding window nth-element queries via circular queue + multiset |
| game_errors.cpp/h | Error state tracking (last code + type); RAII error suppression; global error queue alternative |
| key_definitions.h | Default keyboard-to-action mappings; SDL scancode ΓåÆ player action flags |
| PlayerName.cpp/h | Player name storage and MML parser |
| Scenario.cpp/h | Scenario metadata (name, ID, version, classic flag); version compatibility checks |
| DefaultStringSets.cpp/h | Compiled-in fallback UI strings (25+ categories); TextStrings repository initialization |
| progress.h | Progress dialog API for long-running operations (network, file I/O, uploads) |
| thread_priority_sdl*.cpp/h | Platform-specific thread priority boosting (Windows, macOS, POSIX; dummy fallback) |
| steamshim_child.cpp/h | Child-process side of Steam API shim; named-pipe IPC serialization for achievements/stats/workshop |
| interface_menus.h | Menu ID and item index constants |
| powered_by_alephone*.h | Embedded logo images (BMP/binary resources) |
| AlephSansMono-Bold.h, CourierPrime*.h, ProFontAO.h | Embedded TrueType fonts as static byte arrays |

## Core Responsibilities

- **Game State Machine**: Manage lifecycle transitions (intro ΓåÆ menu ΓåÆ game ΓåÆ replay ΓåÆ epilogue); coordinate high-level transitions, screen fading, color table management
- **User Preferences & Configuration**: Load, persist, and apply settings across graphics (OpenGL, resolution), network (protocol, difficulty), player identity (name, color, team), input controls, audio, environment resources (map, physics, shapes, sounds)
- **Input & Replay System**: Poll keyboard/mouse/joystick input; translate to action flags; queue for networked multiplayer; record game state and input to disk; playback recorded games with speed control and sync validation
- **Logging & Diagnostics**: Thread-safe hierarchical logging with context stacks, severity filtering, file/stderr output; RAII context management; MML configuration
- **Console & Commands**: Interactive console with text editing, command history, macro expansion, command dispatch, kill-message reporting (carnage); MML configuration
- **Achievements & Statistics**: Validate achievements via checksum (prevent custom content exploitation); generate Lua trigger code; Steam integration; asynchronous stats upload via HTTP
- **Action Queueing**: Manage per-player circular buffers for networked input; support zombie controllability; peek/dequeue/enqueue with overflow detection
- **Preferences Dialogs & Widgets**: Bindable preference adapters (type-safe bridge to UI widgets); OpenGL dialog UI; file/directory pickers; crosshair preview; plugin list
- **Player Rendering**: Composite and render 2D player sprites (legs + torso) with state (view angle, color, action, animation frame, brightness)
- **Scenario Selection**: Fullscreen grid UI for scenario browsing; thumbnail loading and scaling; keyboard/mouse/controller input
- **Error Handling**: Global error state tracking; RAII suppression; error code propagation
- **Thread Management**: Platform-specific priority boosting for performance-critical threads; Windows/macOS/POSIX variants with fallback

## Key Interfaces & Data Flow

**Exposed to Other Subsystems:**
- **Preferences API** (`preferences.h`, `preferences.cpp`): Read/write global preference pointers; MML parsing for initialization
- **Logging API** (`Logging.h`): Severity-level log macros; context stack RAII; thread-safe output
- **Console API** (`Console.h`): Register/dispatch commands; input state queries; Lua integration flag
- **Error API** (`game_errors.h`): Set/get/clear error state; RAII suppression scope
- **Action Queues** (`ActionQueues.h`): Enqueue/dequeue/peek player actions; manage zombie state
- **Replay/VBL Timing** (`vbl.h`): Recording setup, header metadata, playback initialization
- **Achievements** (`achievements.h`): Award achievements by key; export Lua trigger code; query disabled status
- **Statistics** (`Statistics.h`): Queue stats; poll upload busy state; wait for completion

**Consumes from Other Subsystems:**
- **Game World** (`map.h`, `world.h`, `player.h`): Current map/physics checksums; player state; dynamic world; local player index
- **Network** (`network.h`): Multiplayer game state; metaserver integration; carnage message config
- **Rendering** (`screen_drawing.h`, `render.h`, `OGL_*`): UI drawing; color tables; screen transitions; shape/texture loading
- **Audio** (`SoundManager.h`, `Music.h`, `OpenALManager.h`): Sound parameters; music control
- **Resources** (`images.h`, `Movie.h`, `FileHandler.h`, `InfoTree.h`, `Plugins.h`): Shape/sound/image loading; file I/O; XML config parsing; plugin management
- **Scripting** (`lua_script.h`, `lua_hud_script.h`): Lua integration for achievements, stats, console
- **Input** (keyboard/mouse/joystick via SDL): Raw input state; scancode processing
- **SDL2 & Platform**: Window events, threading, mutex, dialogs, fonts, surfaces

## Runtime Role

**Initialization Phase:**
- Load default string resources into TextStrings repository
- Initialize preferences from disk (or MML defaults if unavailable)
- Load environment files (map, physics, shapes, sounds) with checksum validation
- Initialize logging system and create log file
- Load scenario metadata and artwork
- Set up input key mappings from preferences
- Initialize achievements singleton and validate against checksums
- Install 30 Hz VBL timer task for input polling

**Frame/Runtime Phase:**
- VBL task polls keyboard/mouse/joystick; converts to action flags; queues to ActionQueues
- Console processes input events (text, editing, command dispatch) when active
- Recording system captures or plays back action flags and game state
- Statistics collector gathers Lua stats and queues for asynchronous upload
- Chat history notifies observers (widgets) of new entries
- Interface state machine processes menu selections and game transitions

**Shutdown Phase:**
- Persist modified preferences to disk
- Flush and close log file
- Complete any pending statistics uploads
- Save/export replay recordings as needed
- Clean up Lua-bound resources (achievements, console contexts)

## Notable Implementation Details

- **Circular Buffer Hierarchy**: `CircularQueue<T>` template provides modular index wrapping for O(1) ops; `CircularByteBuffer` specializes for byte bulk operations with zero-copy support; used by ActionQueues for efficient per-player input buffering
- **RAII Error Suppression**: `ScopedErrorSuppression` class saves/restores error state; scoped exceptions don't leave dangling error codes
- **Thread-Safe Logging**: Per-thread context stacks with mutex protection; LogContext RAII automatically indents nested scopes; lazy initialization of log file on first log call
- **Platform Abstraction for Thread Priority**: Compile-time selection of thread_priority_sdl_*.cpp (Windows/macOS/POSIX); fallback dummy no-op implementation; prevents repeated main-thread priority reductions
- **Steam Shim IPC**: Child process communicates with parent via named pipes (Windows HANDLE / Unix file descriptor); binary serialization for achievements, stats, Workshop queries; avoids direct Steam SDK linking
- **Embedded Resources**: Fonts and images compiled into executable via binary-to-header converters (`bin2h.py`); eliminates runtime file I/O and external dependencies for core UI
- **Preference Binding Pattern**: `Bindable<T>` template subclasses for strings, bools, ints, file paths; bidirectional sync with UI widgets via `Binder<T>` connectors and `BinderSet` batch operations; decouples preference types from UI representation
- **Observer Pattern for Chat**: Chat history maintains vector of entries; `ChatHistoryObserver` interface notifies attached widgets on add/clear; bridges platform-independent data with SDL/Carbon widget implementations
- **Recording Format**: Chunked, run-length-encoded action flags written to disk; metadata header includes player starts, game data, map checksums for sync validation; replay speed controllable independent of original capture rate
- **Scenario Grid Layout**: Dynamic column/row sizing based on window dimensions; scrolling offset to keep selected item visible; thumbnail scaling and caching for performance
