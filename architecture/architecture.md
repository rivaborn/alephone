# Architecture Overview

## Repository Shape

Aleph One is organized into modular subsystems in `Source/`, with supplementary utilities in `Extras/`, `tools/`, and `tests/`, build configuration in `PBProjects/`, and embedded resources (fonts, images) compiled into the binary:

```
alephone/
Γö£ΓöÇΓöÇ Source/
Γöé   Γö£ΓöÇΓöÇ cseries/                   # Platform abstraction layer (types, paths, dialogs, timing, font/color)
Γöé   Γö£ΓöÇΓöÇ files/                     # File I/O, WAD archive handling, resource fork emulation
Γöé   Γö£ΓöÇΓöÇ game_world/                # Entity simulation, physics, AI pathfinding, world state
Γöé   Γö£ΓöÇΓöÇ input/                     # Keyboard/mouse/gamepad input abstraction
Γöé   Γö£ΓöÇΓöÇ lua/                       # Lua scripting subsystem with game world bindings
Γöé   Γö£ΓöÇΓöÇ misc/                      # Preferences, logging, console, achievements, action queueing, UI state
Γöé   Γö£ΓöÇΓöÇ model/                     # 3D skeletal model loading and rendering
Γöé   Γö£ΓöÇΓöÇ network/                   # Peer-to-peer multiplayer networking, metaserver integration
Γöé   Γö£ΓöÇΓöÇ render/                    # Main 3D rendering pipeline (visibility, sorting, rasterization)
Γöé   Γö£ΓöÇΓöÇ screen/                    # 2D display composition, HUD rendering, overhead map
Γöé   Γö£ΓöÇΓöÇ sound/                     # Audio playback, spatial 3D audio, music sequencing
Γöé   Γö£ΓöÇΓöÇ TCPMess/                   # Message-based TCP networking (CommunicationsChannel, MessageDispatcher)
Γöé   Γö£ΓöÇΓöÇ Videos/                    # Recording and playback (Movie, pl_mpeg)
Γöé   Γö£ΓöÇΓöÇ XML/                       # MML parsing and config (InfoTree, XML_MakeRoot, Plugins, XML_LevelScript, QuickSave)
Γöé   Γö£ΓöÇΓöÇ [Interface/Shell/Screen]   # Application lifecycle, rendering, display management
Γöé   ΓööΓöÇΓöÇ [Utilities]                # Steam shim, resource_manager, wad, FileHandler
Γö£ΓöÇΓöÇ Source/Steam/                  # Steam API parent shim (steamshim_parent.cpp)
Γö£ΓöÇΓöÇ tests/                         # Catch2 test suite
Γöé   Γö£ΓöÇΓöÇ main.cpp                   # Test runner
Γöé   ΓööΓöÇΓöÇ replay_film_test.cpp       # Replay determinism validation
Γö£ΓöÇΓöÇ tools/                         # Offline inspection utilities
Γöé   Γö£ΓöÇΓöÇ dumprsrcmap.cpp            # Macintosh resource map inspection
Γöé   ΓööΓöÇΓöÇ dumpwad.cpp                # Marathon WAD file inspection
Γö£ΓöÇΓöÇ Extras/extract/                # Build-time utilities (shape/sound/physics extraction)
ΓööΓöÇΓöÇ PBProjects/                    # Compile-time configuration headers (autoconf-generated feature flags)
```

---

## Major Subsystems

### CSeries

- **Purpose:** Cross-platform abstraction layer shielding the engine from OS-specific APIs; foundational type system, string/path/dialog utilities, timing primitives, and color/font management.
- **Key directories / files:**
  - `cseries.h` ΓÇö Master include header
  - `cstypes.h` ΓÇö Fixed-width integer types, fixed-point arithmetic
  - `BStream.h/cpp` ΓÇö Binary serialization with endianness support
  - `csstrings.h/cpp` ΓÇö Resource string loading, encoding conversion (Mac Roman ΓåÆ UTF-8/UTF-16)
  - `cspaths_sdl.h/cpp` ΓÇö Platform-specific directory resolution
  - `csalerts_sdl.h/cpp` ΓÇö Error dialogs, assertions, debug utilities
  - `csmacros.h` ΓÇö MIN/MAX, clamp, flag manipulation, bounds-checked array access
  - `csmisc_sdl.h/cpp` ΓÇö Tick counting, thread yielding, input polling
  - `mytm_sdl.h/cpp` ΓÇö SDL-based task scheduling, timer callbacks, mutex-protected concurrency
- **Key responsibilities:**
  - Define portable fixed-width integer types and emulate legacy MacOS types (`Rect`, `RGBColor`, `OSErr`)
  - Perform character encoding conversions and byte-order reversal for cross-platform data serialization
  - Resolve OS-specific file paths for application resources, preferences, logs, and save games
  - Display modal alert dialogs and manage dialog widget state
  - Provide monotonic tick counting with drift-free timer task scheduling in SDL threads
  - Supply utility macros for arithmetic, flag testing, and memory operations
- **Key dependencies (other subsystems):**
  - SDL2 (core cross-platform primitives)
  - Standard C++ library (`<chrono>`, `<thread>`, `<atomic>`)

### Files

- **Purpose:** Centralized file I/O abstraction and resource management, including WAD archive format handling, game state persistence, physics data import, binary serialization, and resource fork emulation for macOS compatibility.
- **Key directories / files:**
  - `FileHandler.h/cpp` ΓÇö Cross-platform file I/O abstraction via SDL_RWops; path management; native file dialogs; ZIP archive support
  - `wad.h/cpp` ΓÇö WAD file format reader/writer (versions 0ΓÇô4); multi-version binary parsing; tag-based data extraction; CRC validation
  - `AStream.h/cpp` ΓÇö Templated big-endian/little-endian binary serialization streams
  - `game_wad.h/cpp` ΓÇö Game state persistence: map loading, save game creation/restoration, network map sync
  - `resource_manager.h/cpp` ΓÇö macOS resource fork emulation; AppleSingle/MacBinary parsing
  - `crc.h/cpp` ΓÇö CRC-32 and CRC-16 computation for integrity verification
  - `import_definitions.cpp` ΓÇö Physics definition unpacking from Marathon 1 or WAD formats
  - `WadImageCache.h/cpp` ΓÇö On-disk LRU thumbnail cache with INI metadata persistence
  - `wad_prefs.h/cpp` ΓÇö Preferences persistence via WAD file format with corruption recovery
- **Key responsibilities:**
  - Abstract cross-platform file I/O (open, read, write, seek, close, metadata) over SDL_RWops
  - Parse and write WAD archive format with version evolution support (Marathon 1 ΓÇô Infinity compatibility)
  - Serialize/deserialize full game state (player, monsters, projectiles, effects, platforms, terminals, Lua)
  - Perform big-endian/little-endian byte order conversion with alignment handling
  - Emulate macOS resource forks on non-Mac platforms via format detection and parsing
  - Import and unpack physics definitions for entity types (monsters, weapons, effects, projectiles)
  - Calculate and validate CRC checksums for file integrity
  - Discover files via recursive traversal with typecode filtering (including Steam Workshop)
  - Manage image caching with LRU eviction and resize-on-demand
- **Key dependencies (other subsystems):**
  - GameWorld (entity structures for serialization)
  - Network (map checksum distribution)
  - Lua / XML (script and config loading)
  - Sound / Rendering (audio and visual resource loading)
  - UI/Shell (dialogs and resource strings)

### GameWorld

- **Purpose:** Core simulation engine managing all game entities (player, monsters, projectiles, items, effects, platforms, lights), world geometry (polygon BSP, lines), physics simulation, AI pathfinding, and the main 30 FPS game loop orchestrating frame-by-frame world state updates.
- **Key directories / files:**
  - `map.h/cpp` ΓÇö Core map geometry (polygons, lines, sides, endpoints), entity lists, spatial queries, object lifecycle
  - `marathon2.cpp` ΓÇö Main game loop (`update_world`), world state manager, level transitions, trigger logic
  - `player.h/cpp` ΓÇö Player entity, physics state, inventory, damage/death, powerup management
  - `monsters.h/cpp` ΓÇö Monster AI, behavior state machines, pathfinding, targeting, combat
  - `projectiles.h/cpp` ΓÇö Projectile simulation, ballistics, collision detection, detonation, damage
  - `items.h/cpp` ΓÇö Item lifecycle, placement, pickup mechanics, inventory updates
  - `effects.h/cpp` ΓÇö Visual/audio effects (explosions, splashes, teleportation), animation
  - `platforms.h/cpp` ΓÇö Dynamic platforms/doors, state transitions, collision handling, geometry updates
  - `devices.cpp` ΓÇö Interactive control panels (switches, terminals, refuel, save points)
  - `lightsource.h/cpp` ΓÇö Dynamic lighting, six-phase animation state machines, activation triggers
  - `media.h/cpp` ΓÇö In-world liquids (water, lava, goo, sewage, Jjaro), height calculation, damage
  - `physics.cpp/physics_models.h` ΓÇö Character movement, gravity, collision resolution, aim input
  - `world.h/cpp` ΓÇö Trigonometric tables, point transformations, deterministic RNG
  - `flood_map.h/cpp` ΓÇö Polygon-based graph search for pathfinding
  - `pathfinding.cpp` ΓÇö Waypoint generation, path traversal, random biased paths
  - `interpolated_world.h/cpp` ΓÇö Double-buffered world snapshots, per-frame interpolation (>30 FPS rendering)
  - `weapons.h/cpp` ΓÇö Player weapon state machines, firing, reloading, ammo management
  - `ephemera.h/cpp` ΓÇö Temporary visual objects (particles, sprites), object pool allocation
- **Key responsibilities:**
  - Create, update, and destroy all game entities (player, monsters, projectiles, items, effects) with per-entity state
  - Simulate physics: character movement with gravity/collision, polygon height changes, media interactions at 30 FPS deterministic tick rate
  - Implement monster AI via breadth-first flood-fill pathfinding, target selection, line-of-sight checks, state machines, difficulty scaling
  - Orchestrate main game loop: coordinate all subsystem ticks, manage action queues from player input/Lua/prediction, manage entity lifecycle
  - Manage interactive world elements: platforms (doors, elevators), control panels (terminals, switches), dynamic lights, adjacent geometry cascading
  - Perform collision detection, spatial queries (polygon boundary crossing, line intersections, media containment), visibility checks
  - Execute pathfinding and monster movement: breadth-first flood-fill of polygon graph, waypoint generation, random biased destination selection
  - Manage environmental effects: liquids (height, current, damage), dynamic lighting (intensity animation, state transitions), visual/audio effect spawning
- **Key dependencies (other subsystems):**
  - Rendering (collection loading, shape animation data, camera updates)
  - Audio (play world sounds, 3D positioning, ambient sound updates)
  - Input (player action flags)
  - Network (world state synchronization, client-side prediction rollback)
  - Lua (event callbacks: entity damaged/killed, platform activated, item created)
  - Configuration (MML/XML parsing for entity, weapon, item, platform, light customization)

### Input

- **Purpose:** Abstract SDL2 input devices (keyboard, mouse, gamepad) and convert raw hardware input into scancode-based keymap format; apply user-configured sensitivity and deadzone settings; bridge input events to game actions.
- **Key directories / files:**
  - `joystick.h` ΓÇö Joystick interface declaration; scancode offset constants for controller button/axis mapping
  - `joystick_sdl.cpp` ΓÇö SDL2 game controller implementation; device lifecycle, button/axis translation to keyscans
  - `mouse.h` ΓÇö Mouse input interface declarations; mouselook and button handling
  - `mouse_sdl.cpp` ΓÇö SDL2 mouse implementation; mouselook deltas with sensitivity/acceleration; button and scroll wheel mapping
- **Key responsibilities:**
  - Enumerate and manage SDL2 input device lifecycle (controllers, mouse, hotplug events)
  - Map hardware inputs (buttons, analog sticks, axes, scroll wheel) to scancode-based keymaps
  - Convert continuous analog values (mouse deltas, controller sticks) to discrete key events and aim targeting data
  - Apply user-configured input preferences (sensitivity, deadzone, inversion, key bindings)
  - Handle relative mouse mode for in-game mouselook with acceleration curve
  - Process frame-by-frame input state buffering and device event polling
- **Key dependencies (other subsystems):**
  - SDL2 (device state, events, relative motion tracking)
  - Preferences (key bindings, sensitivity, deadzone, inversion, analog mode flags)
  - GameWorld (player action flags, aim processing)

### RenderMain

- **Purpose:** Orchestrate the rendering pipeline transforming 3D game world geometry and objects into 2D screen output; coordinate visibility determination, polygon sorting, object placement, and rasterization across multiple backends (software rasterizer, OpenGL classic, shader-based GPU rendering).
- **Key directories / files:**
  - `render.h/cpp` ΓÇö Central rendering orchestration: view setup, effect management, pipeline coordination, frame rendering entry point
  - `RenderVisTree.h/cpp` ΓÇö Visibility determination via ray casting to build hierarchical tree of visible polygons
  - `RenderSortPoly.h/cpp` ΓÇö Depth-based polygon sorting with clipping window calculation
  - `RenderPlaceObjs.h/cpp` ΓÇö Object placement, projection, visibility culling, depth-sorting into render tree
  - `RenderRasterize.h/cpp` ΓÇö Polygon clipping and coordinate transformation for rasterization
  - `RenderRasterize_Shader.h/cpp` ΓÇö Shader-based GPU rasterization with multi-pass rendering and post-processing
  - `Rasterizer.h` ΓÇö Abstract rasterizer interface defining rendering contract
  - `Rasterizer_SW.h` ΓÇö Software rasterizer implementation (Scottish Textures)
  - `Rasterizer_OGL.h` ΓÇö OpenGL classic rasterizer adapter
  - `Rasterizer_Shader.h/cpp` ΓÇö Shader-based OpenGL rasterizer with FBO management
  - `scottish_textures.h/cpp` ΓÇö Software texture rasterizer: perspective-correct texture mapping, DDA line tables, transfer modes
  - `AnimatedTextures.h/cpp` ΓÇö Animated wall textures: frame sequencing, tick-based updates, XML configuration
  - `OGL_Render.h/cpp` ΓÇö OpenGL rendering backend: context lifecycle, coordinate transforms, 3D sprite/model rendering
  - `OGL_Setup.h/cpp` ΓÇö OpenGL initialization, extension detection, configuration, resource loading
  - `OGL_Shader.h/cpp` ΓÇö GLSL shader compilation, linking, program management, uniform binding
  - `OGL_FBO.h/cpp` ΓÇö OpenGL framebuffer object management for off-screen rendering and ping-pong buffering
  - `OGL_Textures.h/cpp` ΓÇö OpenGL texture lifecycle: loading, caching, VRAM management, effects (infravision, silhouettes)
  - `OGL_Model_Def.h/cpp` ΓÇö 3D model loading, storage, texture associations, MML configuration
  - `OGL_Subst_Texture_Def.h/cpp` ΓÇö Substitute texture configuration for walls and sprites
  - `OGL_Faders.h/cpp` ΓÇö Fade effect rendering (tint, static, negation, dodge, burn)
  - `ImageLoader.h`, `ImageLoader_SDL.cpp`, `ImageLoader_Shared.cpp` ΓÇö Image file loading (DDS, PNG, BMP), format conversion, mipmap management
  - `Crosshairs.h`, `Crosshairs_SDL.cpp` ΓÇö Crosshair HUD rendering
  - `shapes.cpp` ΓÇö Shape collection loading, bitmap decompression, shading table construction
  - `textures.h/cpp` ΓÇö Bitmap metadata, row address tables, pixel remapping
  - `low_level_textures.h` ΓÇö Template-based texture rasterization for polygons
  - `collection_definition.h` ΓÇö Binary-compatible structures for Marathon shape collections
  - `vec3.h` ΓÇö Vector and matrix types for 3D graphics
- **Key responsibilities:**
  - Initialize and update camera position, orientation, FOV, and screen-to-world projection each frame
  - Build hierarchical visibility tree via ray casting through 2D map geometry from player viewpoint
  - Depth-order visible polygons into rendering tree with clipping window calculation
  - Convert game objects into renderer-ready structures with projection, culling, and depth sorting
  - Clip polygons to viewport, transform to screen space, dispatch to backend rasterizer
  - Abstract rasterizer interface supporting software, OpenGL, and shader-based rendering
  - Handle texture loading, caching, VRAM lifecycle, and animation
  - Apply transfer modes (slides, wobbles, tints, fades), explosions, teleports, camera shake
  - Support loaded 3D models with lighting, skins, and transformations
  - Render weapons, crosshairs, and UI elements in foreground layer
  - Track texture usage, purge unused VRAM, preload textures
- **Key dependencies (other subsystems):**
  - GameWorld (map geometry, player state, object state, entity data)
  - Files (shape/texture loading and caching)
  - Audio (synchronized visual effects)
  - Input (view control during rendering frame)
  - Lua (custom shaders and effects via scripting)
  - CSeries (coordinate types, platform macros)
  - ModelView (3D model rendering)
  - Preferences (rendering backend selection, texture quality, graphics settings)

### RenderOther

- **Purpose:** Handle all 2D screen composition, UI rendering, and visual effects; manage HUD, overhead map, text rendering, screen fades, and 2D asset blitting across multiple rendering backends (SDL2 software, OpenGL, Lua scripting).
- **Key directories / files:**
  - `screen.h/cpp` ΓÇö Central display and window lifecycle; surface allocation; frame composition orchestration; OpenGL setup
  - `HUDRenderer.h/cpp` ΓÇö Base HUD rendering class; weapon/ammo/shield display; interface state management
  - `HUDRenderer_OGL.h/cpp` ΓÇö OpenGL-accelerated HUD implementation with motion sensor, text, and shape rendering
  - `HUDRenderer_SW.h/cpp` ΓÇö Software (SDL-based) HUD implementation for CPU rendering path
  - `HUDRenderer_Lua.h/cpp` ΓÇö Lua-scripted HUD support with blip tracking and masking modes
  - `game_window.h/cpp` ΓÇö HUD buffering, weapon interface layouts, MML customization, motion sensor initialization
  - `screen_drawing.h/cpp` ΓÇö SDL2 primitive rendering (shapes, text, polygons, lines, rectangles) and port abstraction
  - `OverheadMapRenderer.h/cpp` ΓÇö Base overhead map (automap) system; world-to-screen transforms; entity and annotation rendering
  - `OverheadMap_OGL.h/cpp` ΓÇö OpenGL automap implementation with geometry batching and cached rendering
  - `OverheadMap_SDL.h/cpp` ΓÇö SDL software automap implementation
  - `motion_sensor.h/cpp` ΓÇö Radar/motion sensor display with entity tracking, blip rendering, and network compass
  - `computer_interface.h/cpp` ΓÇö Terminal/computer interface rendering, text compilation, input handling, serialization
  - `fades.h/cpp` ΓÇö Screen fade effects, color tinting, damage feedback, gamma correction, MML configuration
  - `FontHandler.h/cpp` ΓÇö Font specification, metric calculation, OpenGL texture atlas generation, dual-mode text rendering
  - `images.h/cpp` ΓÇö PICT resource loading/decompression, color table management, picture scaling and tiling
  - `Image_Blitter.h/cpp` ΓÇö 2D image rendering with scaling, cropping, tinting
  - `OGL_Blitter.h/cpp` ΓÇö OpenGL texture tiling and efficient 2D image rendering via GPU textures
  - `Shape_Blitter.h/cpp` ΓÇö Shapes collection bitmap rendering with transformations (scale, rotate, tint)
  - `ViewControl.h/cpp` ΓÇö FOV/camera parameters, landscape/sky rendering config, teleport effects, on-screen font management
  - `ChaseCam.h/cpp` ΓÇö Third-person behind-view camera; physics-based damping; wall collision detection
  - `OGL_LoadScreen.h/cpp` ΓÇö Load screen display with progress bars and image scaling
  - `TextStrings.h/cpp` ΓÇö Localized text string repository; UTF-8 conversion; MML parsing
  - `TextLayoutHelper.h/cpp` ΓÇö Non-overlapping rectangle reservation for UI layout
  - `screen_shared.h` ΓÇö Global screen state (color tables, on-screen messages, FPS display, visual effects)
- **Key responsibilities:**
  - Initialize and manage the game window, screen mode detection, surface allocation, and OpenGL context creation with fallback to software rendering
  - Render the heads-up display (HUD) each frame with support for multiple backend implementations (SDL software, OpenGL hardware, Lua scripting)
  - Compose the complete display each frame by blitting the 3D world view, overlaying HUD, terminal interfaces, overhead map, and screen effects
  - Manage image and shape resource loading from WAD/resource files with color table handling and format conversion
  - Render 2D primitive geometry (shapes, text, rectangles, lines, polygons) with SDL2 or OpenGL, supporting clipping and alignment
  - Maintain the overhead map (automap) system with flood-fill visibility algorithms for checkpoint maps and dual rendering paths
  - Apply screen effects including fades, color tints, damage feedback, gamma correction, and teleport visual transitions
  - Provide text rendering with TrueType font support, bitmap fonts, scaling, word-wrap, and multi-backend abstraction
  - Manage viewport and scissor configuration, coordinate transformation, and port-based drawing abstraction for flexible render target redirection
- **Key dependencies (other subsystems):**
  - RenderMain (3D camera parameters, world view output, view frustum)
  - GameWorld (game world state: player data, entity positions, map geometry)
  - Files (resource loading via FileHandler and LoadedResource)
  - Audio (HUD button click effects via SoundManager)
  - Lua (HUD callbacks L_Call_HUDDraw, L_Call_Terminal_Exit)
  - Network (multiplayer HUD: compass, team data)
  - Preferences (graphics settings, bit depth, MML customizations)
  - CSeries (coordinate types, platform text encoding)

### ModelView

- **Purpose:** Load, store, and render 3D skeletal models in multiple file formats (Dim3 XML, 3D Studio Max, Wavefront OBJ); provide unified internal representation, skeletal animation playback with frame interpolation, and OpenGL rendering with multi-pass shader support.
- **Key directories / files:**
  - `Model3D.h/cpp` ΓÇö Model3D data structures (vertex geometry, skeletal animation, bones, frames, sequences, bounding boxes, tangent vectors); skeletal animation implementation (bone matrices, vertex blending, frame interpolation)
  - `Dim3_Loader.h/cpp` ΓÇö Load 3D skeletal models from Dim3 XML format; parse geometry, bones, rigging, animation data; topological bone sorting
  - `StudioLoader.h/cpp` ΓÇö Load 3D Studio Max .3ds files; parse hierarchical chunk format; extract vertex positions, texture coordinates, polygon indices
  - `WavefrontLoader.h/cpp` ΓÇö Load Wavefront OBJ files; parse geometry; handle vertex deduplication, fan tessellation, coordinate system conversion
  - `ModelRenderer.h/cpp` ΓÇö OpenGL rendering with multi-pass shaders, depth-sorting, and texture/lighting callbacks
- **Key responsibilities:**
  - Load 3D models from Dim3 XML, 3D Studio Max (.3ds), and Wavefront OBJ file formats
  - Parse geometry (vertex positions, normals, texture coordinates), skeletal data (bones, hierarchy, vertex weights), and animation sequences
  - Compute skeletal animation via bone matrix transforms, vertex blending across dual-bone influences, and frame interpolation with angle blending
  - Generate and normalize vertex normals with flat, smooth, and split-by-threshold modes
  - Compute tangent and bitangent vectors for normal mapping
  - Perform coordinate system conversions (right-handed to left-handed) across loaders
  - Render 3D models to OpenGL with multi-pass shader support, texture binding, external lighting
  - Implement centroid-based depth-sorting for polygon rendering when Z-buffer unavailable
  - Optimize rendering by batching separable shaders and caching sorted indices and lighting colors
- **Key dependencies (other subsystems):**
  - Files (file path resolution via FileSpecifier; resource loading)
  - Configuration (XML/MML parsing via InfoTree)
  - GameWorld (trigonometric tables, angle macros, positioning primitives)
  - CSeries (platform macros, memory operations)
  - Rendering (OpenGL types, shader compilation, texture management)

### Sound

- **Purpose:** Provide comprehensive audio playback for the Aleph One game engine, managing sound effects, music sequences, and 3D spatial audio through an OpenAL backend; handle audio file decoding (WAV, FLAC, OGG via libsndfile), playback prioritization, real-time parameter updates, and listener-based attenuation.
- **Key directories / files:**
  - `OpenALManager.h/cpp` ΓÇö Device/context initialization, audio player lifecycle, source pooling, 3D listener updates
  - `SoundManager.h/cpp` ΓÇö Central sound effect management, playback, spatial parameter calculation, ambient sound coordination
  - `SoundPlayer.h/cpp` ΓÇö Per-instance sound playback with 3D positioning, volume transitions, rewind handling
  - `MusicPlayer.h/cpp` ΓÇö Music playback with segment-based composition, crossfading, fade effects, sequence transitions
  - `Music.h/cpp` ΓÇö Music slot management (intro/level), playlist handling, fade orchestration, game state tracking
  - `AudioPlayer.h/cpp` ΓÇö Base class for OpenAL source lifecycle, buffer queueing, format tracking, thread-safe parameter updates
  - `StreamPlayer.h/cpp` ΓÇö Callback-driven streaming audio for dynamic sources (intro videos)
  - `Decoder.h/cpp` ΓÇö Abstract decoder interface for pluggable audio format support
  - `SndfileDecoder.h/cpp` ΓÇö Concrete libsndfile-based decoder with SDL_RWops file abstraction
  - `SoundFile.h/cpp` ΓÇö M1/M2 sound file parsing, System 7 header handling, permutation loading
  - `ReplacementSounds.h/cpp` ΓÇö External sound file loading and override registry
  - `SoundsPatch.h/cpp` ΓÇö Binary sound patch loading and patched sound definition lookup
- **Key responsibilities:**
  - Initialize/shutdown OpenAL device, context, and maintain a pooled set of audio sources with priority-based allocation
  - Load audio files via pluggable decoders (libsndfile), parse M1/M2 sound file formats, and manage in-memory sound caches with configurable memory budgets
  - Play individual sound effects with 3D spatial positioning, distance-based attenuation, obstruction/muffling effects, and smooth parameter transitions
  - Manage music playback via MusicPlayer instances organized into sequences with support for segment composition, crossfading, and fade-in/fade-out transitions
  - Calculate dynamic audio parameters (volume, stereo panning, pitch, obstruction flags) based on listener/source position and world state
  - Update 3D listener position each frame and manage ambient sound sources with per-frame parameter recalculation
  - Support sound replacement/patching from external audio files (MML configuration) and binary patch streams without code recompilation
  - Coordinate thread-safe parameter updates between game thread and audio callback thread using lock-free data structures
- **Key dependencies (other subsystems):**
  - GameWorld (listener position/orientation, obstruction queries, ambient sound enumeration)
  - Files (FileSpecifier, OpenedFile, LoadedResource, resource fork emulation)
  - Configuration (MML/XML parsing for sound patches)
  - Lua (event triggers, custom audio behavior via scripting)
  - Misc (game state queries for music transitions)

### Network

- **Purpose:** Implement multiplayer networking enabling peer-to-peer and hub-and-spoke game topologies with real-time synchronization, metaserver integration for game discovery, and message routing across networked players.
- **Key directories / files:**
  - `network.h/cpp` ΓÇö Core multiplayer API (player gathering/joining, connection lifecycle, state machine)
  - `network_messages.h/cpp` ΓÇö Game protocol message serialization (join handshake, topology, map/physics distribution)
  - `network_star_hub.cpp` ΓÇö Hub-side star topology (collects player actions, distributes synchronized state)
  - `network_star_spoke.cpp` ΓÇö Client-side star topology (sends actions, receives hub state, maintains timing)
  - `StarGameProtocol.h/cpp` ΓÇö Game protocol interface adapter for star topology
  - `network_games.h/cpp` ΓÇö Multiplayer game mode logic (scoring, rankings, game-over detection)
  - `network_metaserver.h/cpp` ΓÇö Metaserver client (login, game listing, chat relay, player list sync)
  - `network_dialogs.h/cpp` ΓÇö UI dialogs for game gathering, joining, setup configuration
  - `ConnectPool.h/cpp` ΓÇö Async TCP connection pooling with background threads for DNS resolution
  - `NetworkInterface.h/cpp` ΓÇö ASIO-based socket abstraction (TCP/UDP, IP address handling)
  - `network_udp.cpp` ΓÇö UDP socket lifecycle and async receive thread
  - `SSLP_limited.cpp/SSLP_API.h` ΓÇö LAN service discovery (Simple Service Location Protocol)
  - `Pinger.h/cpp` ΓÇö ICMP ping measurement for latency monitoring
  - `HTTP.h/cpp` ΓÇö libcurl-based HTTP client for metaserver and update checks
  - `PortForward.h/cpp` ΓÇö UPnP port forwarding for NAT traversal
  - `StandaloneHub/` ΓÇö Dedicated network hub server (gatherer relay)
- **Key responsibilities:**
  - Accept player connections, negotiate capabilities, manage client lifecycle (connecting ΓåÆ connected ΓåÆ in-game)
  - Deserialize incoming network messages, dispatch to handlers, queue outgoing messages (chat, topology, game state)
  - Maintain and broadcast player list and game state to all connected players
  - Compress and stream map data, physics scripts, and embedded Lua to joiners; accumulate received data on joiner side
  - Implement star protocol: hub collects action flags from all spokes, calculates timing offsets, synthesizes flags for lagging players, broadcasts synchronized frames
  - Advertise local games to metaserver and LAN via SSLP; browse and join remote games
  - Handle metaserver login (plaintext, XOR, HTTPS key exchange) and maintain authenticated session
  - Measure per-player latency and jitter via ping; track packet loss and statistics
  - Spawn background threads for async DNS resolution and TCP handshake without blocking game loop
- **Key dependencies (other subsystems):**
  - GameWorld (dynamic world state, map polygons, player data, physics, embedded Lua scripts)
  - Preferences (network settings, gather port, login credentials, muting rules)
  - UI/Misc (dialogs, player selection, chat input)
  - System (time, filesystem, preferences persistence)

### TCPMess

- **Purpose:** Reliable, message-based TCP networking for peer-to-peer multiplayer communication. Manages bidirectional socket connections, asynchronous/synchronous message queueing and dispatch, serialization/deserialization with optional compression, and handler-based message routing by type.
- **Key directories / files:**
  - `CommunicationsChannel.h/cpp` ΓÇö TCP connection lifecycle, message queue management, async pump and blocking receive APIs
  - `Message.h/cpp` ΓÇö Abstract message interface; serialization/deserialization bridges `AIStreamBE`/`AOStreamBE` and raw buffer formats
  - `MessageHandler.h` ΓÇö Abstract handler interface with function pointer and member method templates
  - `MessageDispatcher.h` ΓÇö Type-ID message router with fallback support
  - `MessageInflater.h/cpp` ΓÇö Prototype-pattern registry for message deserialization and instantiation
- **Key responsibilities:**
  - Establish, maintain, and tear down TCP socket connections
  - Queue and transmit outgoing messages with 8-byte header + body framing; batch-flush multiple channels
  - Receive, buffer, and parse incoming messages with header validation and timeout enforcement
  - Serialize/deserialize messages with optional inflation (decompression)
  - Route messages to registered handlers based on message type ID
  - Provide synchronous blocking receive with overall and inactivity timeout tracking
  - Support message type filtering, cloning, and prototype-based instantiation
- **Key dependencies (other subsystems):**
  - Network (socket primitives via `NetworkInterface`: `TCPsocket`, `TCPlistener`, `IPaddress`)
  - Files (stream-based serialization via `AStream`: `AIStreamBE`, `AOStreamBE`)
  - CSeries (timing utilities: `machine_tick_count()`, `sleep_for_machine_ticks()`)

### Lua

- **Purpose:** Provide scripting capabilities enabling map creators and modders to customize game behavior; manage script loading, execution, event dispatch, and state serialization; expose game world objects (entities, map geometry, player state, audio, HUD) to Lua via C API binding layer.
- **Key directories / files:**
  - `lua_script.h/cpp` ΓÇö Core script loading, execution, event dispatch, state management
  - `lua_templates.h` ΓÇö Template classes for binding C++ objects to Lua as userdata
  - `lua_serialize.h/cpp` ΓÇö Binary serialization/deserialization of Lua state
  - `lua_mnemonics.h` ΓÇö String-to-integer mnemonic lookup tables for game constants
  - `lua_map.h/cpp`, `lua_monsters.h/cpp`, `lua_objects.h/cpp`, `lua_player.h/cpp`, `lua_projectiles.h/cpp` ΓÇö Bindings for map geometry, monsters, items, player, projectiles
  - `lua_hud_objects.h/cpp`, `lua_hud_script.h/cpp` ΓÇö HUD element bindings and script lifecycle
  - `lua_music.h/cpp`, `lua_ephemera.h/cpp` ΓÇö Music and temporary effect bindings
  - `lua.h`, `lauxlib.h`, `lualib.h`, `luaconf.h` ΓÇö Lua 5.2 core interpreter headers
- **Key responsibilities:**
  - Load Lua scripts from map files, plugins, and memory buffers; initialize VM state with engine functions and sandbox configuration
  - Route game events (player actions, damage, kills, item pickup, terminal interaction, level state changes) to registered Lua callbacks
  - Expose map entities, geometry, physics objects, and game state to Lua through typed bindings with getter/setter methods
  - Serialize/deserialize Lua objects to binary format with reference deduplication for game save/load
  - Manage HUD script lifecycle (init, draw, resize, cleanup) with callbacks for custom interface overlays
  - Enforce read-only vs. read-write access to game systems via `LuaMutabilityInterface`
  - Route Lua-requested actions (weapon change, camera animation, console output) into game engine via action queues
  - Provide symbolic name-to-code mappings for weapons, items, monsters, damage types, game modes
- **Key dependencies (other subsystems):**
  - GameWorld (entity state queries and modifications, event dispatch)
  - Files (script loading from WAD/plugin resources)
  - Network (multiplayer game state, netscript contexts)
  - Audio (music system control)
  - Rendering (HUD rendering callbacks, camera animation)
  - Configuration (MML/XML parsing for Lua customization)
  - Input (player action injection)

### XML

- **Purpose:** Centralized configuration, metadata, and scripting management via Marathon Markup Language (MML) XML files. Parses MML to populate game subsystems, orchestrates level-specific scripts (MML, Lua, movies, music), manages plugins with resource patch loading, and handles quick save file metadata.
- **Key directories / files:**
  - `InfoTree.cpp/h` ΓÇö Type-safe Boost.PropertyTree wrapper; XML/INI parsing with game-type serialization
  - `XML_MakeRoot.cpp` ΓÇö Root MML dispatcher; routes subsystem elements to ~60 `parse_mml_*()` handlers
  - `XML_ParseTreeRoot.h` ΓÇö Public MML API (`ParseMMLFromFile`, `ParseMMLFromData`, `ResetAllMMLValues`)
  - `Plugins.cpp/h` ΓÇö Plugin discovery, validation, resource loading (MML, shapes, sounds, maps), exclusivity enforcement
  - `XML_LevelScript.cpp/h` ΓÇö Per-level script orchestration; executes MML/Lua, manages movies, music, load screens
  - `QuickSave.cpp/h` ΓÇö Quick save metadata, preview image caching, UI dialogs, LRU cache
- **Key responsibilities:**
  - Parse XML/INI configuration files via Boost.PropertyTree with bounds checking and enum validation
  - Load and reset MML configurations during engine initialization and level transitions
  - Route parsed XML elements to subsystem-specific parsers (~60 handlers for weapons, monsters, items, physics, interface, etc.)
  - Discover and validate plugins from directories and Steam workshop
  - Load plugin resources and enforce mutual exclusivity (single HUD Lua, theme, stats Lua)
  - Execute level-specific scripts, manage movie playback, coordinate music playlists, configure load screens
  - Manage quick save files with metadata (level name, playtime, player count) and preview images
  - Serialize/deserialize game-specific numeric types (fixed-point angles, world units, colors) between XML and memory
- **Key dependencies (other subsystems):**
  - Files (cross-platform file I/O via `FileSpecifier`, `DirectorySpecifier`, `OpenedFile`)
  - GameWorld types (`shape_descriptor`, `damage_definition`, `angle`, `_fixed`, `RGBColor`, `world_point2d/3d`)
  - Subsystem-specific handlers (~60 `parse_mml_*()` and `reset_mml_*()` functions for weapons, monsters, items, interface, physics, etc.)
  - Lua (level script execution)
  - Audio/Music (level script execution)
  - Scenario (active map metadata and resource context)
  - Preferences (solo Lua, stats, network mode flags)
  - External: Boost.PropertyTree, Boost.Iostreams, Steam workshop API

### Misc

- **Purpose:** Foundational utilities, configuration, and infrastructure services aggregating preferences, logging, console commands, replay recording/playback, achievements, input action queueing, statistics collection, and embedded resources (fonts, images); acts as bridge between core game systems and platform implementations.
- **Key directories / files:**
  - `interface.h/cpp` ΓÇö Game state machine (intro, menus, gameplay, replays, epilogue); transitions and screen fading
  - `preferences.h/cpp` ΓÇö Complete preferences/configuration system (graphics, network, player, audio, input, environment)
  - `vbl.h/cpp` ΓÇö Input polling, action flag capture, replay recording/playback, VBL 30 Hz timer task
  - `Logging.h/cpp` ΓÇö Thread-safe logging with context stacks, severity levels, file/stderr output
  - `Console.h/cpp` ΓÇö Interactive console with text input, command execution, history, macros, kill-messages
  - `achievements.h/cpp` ΓÇö Achievement validation, Lua trigger generation, Steam integration
  - `ActionQueues.h/cpp` ΓÇö Multi-player circular-buffer action flag queues with zombie state
  - `Statistics.h/cpp` ΓÇö Gameplay stats collection from Lua; asynchronous HTTP POST upload
  - `preference_dialogs.h/cpp` ΓÇö OpenGL graphics preferences dialog UI
  - `binders.h` ΓÇö State synchronization protocol for bidirectional UI widget binding
  - `CircularQueue.h`, `CircularByteBuffer.h/cpp` ΓÇö Template circular FIFO and byte specialization
  - `Random.h` ΓÇö PRNG implementations (Marsaglia KISS, MWC, LFIB4); independent instances per subsystem
  - `game_errors.h/cpp` ΓÇö Error state tracking; RAII error suppression
  - `key_definitions.h` ΓÇö Default keyboard-to-action mappings
  - `Scenario.h/cpp` ΓÇö Scenario metadata, version compatibility checks
  - `DefaultStringSets.h/cpp` ΓÇö Compiled-in fallback UI strings; TextStrings repository initialization
  - `AlephSansMono-Bold.h`, `CourierPrime*.h`, `ProFontAO.h` ΓÇö Embedded TrueType fonts as static byte arrays
  - `powered_by_alephone*.h` ΓÇö Embedded logo images (BMP/binary resources)
- **Key responsibilities:**
  - Manage game state machine lifecycle transitions (intro ΓåÆ menu ΓåÆ game ΓåÆ replay ΓåÆ epilogue); coordinate high-level transitions, screen fading, color table management
  - Load, persist, and apply settings across all subsystems (graphics, network, player identity, input controls, audio, environment resources)
  - Poll keyboard/mouse/joystick input; translate to action flags; queue for networked multiplayer; record/playback recorded games with speed control and sync validation
  - Provide thread-safe hierarchical logging with context stacks, severity filtering, file/stderr output, RAII context management
  - Implement interactive console with text editing, command history, macro expansion, command dispatch
  - Validate achievements via checksum; generate Lua trigger code; Steam integration; asynchronous stats upload via HTTP
  - Manage per-player circular buffers for networked input; support zombie controllability; detect overflow
  - Bind preferences to UI widgets bidirectionally via `Bindable<T>` and `Binder<T>` adapters
  - Composite and render 2D player sprites (legs + torso) with state (view angle, color, action, animation)
  - Provide fullscreen scenario grid UI for scenario browsing; thumbnail loading and scaling
  - Track global error state; RAII suppression scopes; error code propagation
  - Boost thread priority for performance-critical threads on multiple platforms (Windows/macOS/POSIX)
- **Key dependencies (other subsystems):**
  - GameWorld (current map/physics checksums, player state, dynamic world state)
  - Network (multiplayer game state, metaserver integration, carnage message config)
  - Rendering (UI drawing, color tables, screen transitions, shape/texture loading)
  - Audio (sound parameters, music control)
  - Files (shape/sound/image loading, file I/O, XML config parsing, plugin management)
  - Lua (achievements, stats, console integration)
  - Input (raw input state, scancode processing)
  - SDL2 & Platform (window events, threading, dialogs, fonts)

### Videos

- **Purpose:** Gameplay recording and video playback. Encodes game footage and audio into WebM files via background thread processing. Decodes MPEG-1 video and MP2 audio for in-game video playback, supporting both hardware-accelerated (OpenGL) and software rendering paths.
- **Key directories / files:**
  - `Movie.h/cpp` ΓÇö Singleton interface for recording state, frame queueing, background thread coordination, VP8/Vorbis encoding to WebM
  - `pl_mpeg.h` ΓÇö Self-contained MPEG-1 decoder with Program Stream demuxing, frame-accurate seeking, MP2 audio support
- **Key responsibilities:**
  - Capture video frames from OpenGL framebuffer or SDL software surfaces
  - Encode video frames to VP8 codec with configurable bitrate and quality scaling by resolution
  - Encode audio to Vorbis codec with OpenALManager loopback capture
  - Multiplex audio and video into WebM/Matroska container with frame synchronization
  - Manage thread-safe producerΓÇôconsumer pipeline (main thread captures, encode thread processes asynchronously)
  - Decode MPEG-1 video frames and MP2 audio with PTS timestamp handling
  - Convert between color spaces (ARGBΓåÆI420 YUV, YCrCbΓåÆRGB) for encoding and display
  - Support frame-accurate seeking, frame-skip modes, and audio lead-time management for A/V sync
- **Key dependencies (other subsystems):**
  - RenderMain / RenderOther (rendered frame buffers: OpenGL framebuffer objects or SDL surfaces)
  - Screen (viewport information, surface access, OpenGL detection state)
  - Sound (OpenALManager for audio device loopback capture for recording)
  - Misc (FPS target, input state, user alerts, file dialogs)
  - External: libvpx, libvorbis, libmatroska, libyuv, SDL2, OpenGL

### Steam

- **Purpose:** Steam API parent shim process. Acts as a bridge between the Steamworks SDK and the game child process. Initializes the Steam SDK, spawns and manages the game as a child process, and relays Steam API calls, callbacks, and workshop operations through bidirectional pipes using a binary protocol.
- **Key directories / files:**
  - `Source/Steam/steamshim_parent.cpp` ΓÇö Parent process lifecycle, Steam SDK initialization, child process spawning, binary protocol command parsing, async callback relay
- **Key responsibilities:**
  - Spawn child game process with platform-specific mechanisms (Windows `CreateProcessW` / Unix `fork`+`execvp`)
  - Establish bidirectional pipe communication with game child
  - Initialize and maintain Steamworks SDK interface pointers and state
  - Parse and dispatch binary protocol commands from child process to appropriate Steam API methods
  - Handle Steamworks async callbacks and relay results back to child
  - Manage workshop item lifecycle operations (upload, update, query, delete)
  - Persist achievement and statistics data through Steam UGC/UserStats APIs
  - Monitor game overlay activation state
  - Set up environment variables for child process Steam runtime integration
- **Key dependencies (other subsystems):**
  - Game shell (child process command-line parsing, initialization, main event loop)
  - Steamworks SDK (Steam API callbacks, async results, workshop metadata, user statistics)
  - Platform I/O (Windows pipes: `CreatePipe`, `WriteFile`, `ReadFile`; POSIX pipes: `pipe`, `fork`, `execvp`)
  - Boost.Filesystem (program binary location, workshop directory path resolution)

### tests

- **Purpose:** Testing infrastructure for Aleph One via Catch2 test framework. Validates replay film loading, playback, and random seed consistency to ensure replay determinism across engine updates.
- **Key directories / files:**
  - `tests/main.cpp` ΓÇö Catch2 test runner entry point; parses and filters engine shell options before framework initialization
  - `tests/replay_film_test.cpp` ΓÇö Replay film validation suite; loads, executes, and verifies replay seeding and determinism
- **Key responsibilities:**
  - Parse and filter engine-specific shell options from command-line arguments before Catch2 initialization
  - Load replay files from configurable directory trees and validate format integrity
  - Execute replays at maximum speed and verify random seed consistency against filename expectations
  - Extract and validate random number state from replay execution
  - Provide auto-seeding utility to rename and seed malformed replay files
  - Manage test-wide application lifecycle (initialize/shutdown)
- **Key dependencies (other subsystems):**
  - Game shell (application lifecycle: `initialize_application()`, `shutdown_application()`, `main_event_loop()`, `handle_open_document()`)
  - GameWorld (random number state: `set_random_seed()`, `get_random_seed()`, `global_random()`)
  - Files (cross-platform file I/O: `FileSpecifier`, `dir_entry`)
  - Misc (replay control: `set_replay_speed()`, shell options struct with `replay_directory` field)
  - Preferences (graphics preferences reference)

### tools

- **Purpose:** Command-line utilities for offline inspection and debugging of Marathon/Aleph One data files and binary resources. Standalone programs that analyze WAD files and Macintosh resource maps, extracting and displaying file structure metadata in human-readable format.
- **Key directories / files:**
  - `tools/dumprsrcmap.cpp` ΓÇö Utility to parse and display Macintosh resource map metadata (types, IDs, sizes)
  - `tools/dumpwad.cpp` ΓÇö Utility to parse and display Marathon WAD file structure, headers, directories, and level metadata
- **Key responsibilities:**
  - Parse command-line arguments and validate file input paths
  - Open and read Marathon data files (resource forks; WAD files)
  - Parse binary file structures (resource maps, WAD headers, directory entries, level metadata)
  - Extract and interpret resource/level metadata (types, IDs, sizes, mission/environment flags, entry points)
  - Enumerate data elements (resource types/IDs; WAD tags/entries)
  - Decode and display human-readable flag interpretations
  - Output inspection results to stdout in formatted text
- **Key dependencies (other subsystems):**
  - Files (resource I/O: `open_res_file()`, `get_resource_id_list()`, `get_resource()`, `count_resources()`)
  - Files (WAD I/O: `open_wad_file_for_reading()`, `read_wad_header()`, `read_directory_data()`)
  - Files (FileHandler_SDL: `OpenedFile`, `FileSpecifier`)
  - CSeries (serialization utilities: `StreamToValue()`, `StreamToBytes()` via `Packing`)
  - External: SDL2 (`SDL_RWops`)

### Extras

- **Purpose:** Provide off-line build-time utilities extracting and converting Marathon's legacy Macintosh resource data into consolidated binary formats usable by the engine; includes shape geometry extraction, sound resource consolidation, and physics WAD patch generation.
- **Key directories / files:**
  - `Extras/extract/shapeextract.cpp` ΓÇö Extract shape resources from Macintosh .256 resource files; write consolidated binary shape collections with embedded offset tables
  - `Extras/extract/sndextract.cpp` ΓÇö Consolidate sound resources ('snd ') from one or more Macintosh resource files with permutation support into single binary sound definition file
  - `Extras/physics_patches.cpp` ΓÇö Generate delta WAD patch files by byte-level comparison of original and modified physics WAD files with checksum linkage
- **Key responsibilities:**
  - Extract shape geometry data from Macintosh resources (IDs 128ΓÇô255, 1128ΓÇô1255) with computed offset and length tracking
  - Consolidate sound resources with permutation support and parse legacy sound header formats (standard and extended)
  - Calculate byte-aligned offsets and sizes for efficient engine resource loading
  - Perform byte-by-byte comparison of physics WAD files to identify and isolate changed regions
  - Write consolidated binary output files with collection/definition headers and embedded metadata tables
  - Manage Macintosh resource file I/O and handle lifecycle (OpenResFile, HLock, ReleaseResource, CloseResFile)
  - Implement fallback sound inheritance (secondary resource sources inherit missing sounds from primary source)
  - Generate patches with parent file checksum linkage for validation
- **Key dependencies (other subsystems):**
  - Files (WAD file format reading, binary serialization)
  - CSeries (endianness conversion, memory operations)

### PBProjects

- **Purpose:** Provide build-time configuration headers declaring compile-time feature flags, library availability, and platform-specific paths for the Aleph One game engine; control which optional subsystems, external dependencies, and platform features are compiled into the executable.
- **Key directories / files:**
  - `config.h` ΓÇö Auto-generated configuration header from autoconf declaring compile-time feature flags and library availability
  - `confpaths.h` ΓÇö Configuration header defining platform-specific resource directory paths (currently inactive)
- **Key responsibilities:**
  - Declare feature detection flags for optional libraries (Boost, SDL, OpenGL, audio codecs, compression)
  - Check availability of standard POSIX and C library headers
  - Identify platform and version information for conditional compilation
  - Control compilation of optional subsystems (CURL, miniupnpc, zlib, libpng)
  - Define platform-specific resource file paths (intended; currently disabled)
  - Configure runtime resource lookup behavior (Lua integration flags)
- **Key dependencies (other subsystems):**
  - All subsystems (via `#include "config.h"` throughout codebase)

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
   - **CSeries Setup** ΓÇö Initialize platform abstraction layer: allocate global tick counter task, resolve application directory paths, initialize platform-specific alert/dialog handling
   - **Preferences & Logging** ΓÇö Load preferences from disk (or MML defaults); initialize logging system; set up error handling
   - **File System & Resources** ΓÇö Load default resource files (maps, physics, shapes, sounds) with checksum validation; initialize resource file stack
   - **XML subsystem** invokes `ParseMMLFromFile()` to load default MML configurations:
     - Routes parsed elements to ~60 subsystem-specific `parse_mml_*()` handlers
     - Populates weapons, monsters, items, physics, interface, shapes, damage definitions, etc.
   - **Plugins subsystem** discovers and validates plugins from directories and Steam workshop:
     - Loads plugin MML, resource patches (shapes, sounds, maps)
     - Enforces mutual exclusivity (single HUD Lua, theme, stats Lua)
   - **GameWorld** ΓÇö Allocate entity storage arrays, load default physics definitions, initialize deterministic RNG seeds, prepare world state for first level load
   - **Rendering Setup** ΓÇö Allocate visibility tree and rendering data structures; initialize rasterizer backend (software/OpenGL/shader); load shape collections and texture managers; compile GLSL shaders (if GPU rendering enabled)
   - **Audio System** ΓÇö Initialize OpenAL device and context; allocate source pool; load sound file format negotiation with SDL
   - **Input System** ΓÇö Enumerate SDL2 input devices (keyboard, mouse, gamepad), establish event polling, enable relative mouse mode
   - **Videos subsystem** initializes OpenGL detection and color space conversion tables
   - **Network (if Multiplayer)** ΓÇö Initialize socket layer, spawn connection listener (for hosting) or prepare connection state (for joining), load metaserver client; **TCPMess** registers message prototypes via `MessageInflater` for netgame
   - **Lua (Optional)** ΓÇö Create Lua VM contexts (embedded scripts, solo level, netscript if applicable), register engine C API bindings, load and execute startup scripts
   - **Main Loop** ΓÇö Install VBL 30 Hz timer task for input polling and game ticks; begin rendering frame loop; transition to menu or directly to gameplay based on command-line args

3. **Per-level initialization**:
   - **XML_LevelScript** loads and executes level-specific scripts from map resource 128:
     - Parses level MML, executes Lua scripts (level-specific)
     - Configures movies, music playlists, load screens

### Per-frame / Main Loop

**~30 FPS Game Tick (Deterministic Physics/AI/Logic):**
1. **Input Poll (VBL Task)** ΓÇö SDL poll keyboard/mouse/gamepad events; convert to action flags; queue to ActionQueues for networked players or apply to local player
2. **Network Sync (if Multiplayer)** ΓÇö Hub/spoke protocol: retrieve peer action flags, calculate timing offsets, broadcast synchronized frame; client-side prediction rollback if needed; **TCPMess** calls `pump()` on all active `CommunicationsChannel` instances to process queued incoming/outgoing messages asynchronously and route via `MessageDispatcher`
3. **Game World Update** (`update_world` in GameWorld) ΓÇö Execute main simulation loop at 30 FPS:
   - Process action queues (real player, Lua injection, prediction systems); merge and dequeue
   - Update physics: player movement, gravity, collision resolution via `physics.cpp`
   - Update entities: monster AI/pathfinding, projectile ballistics, item animation, effect updates
   - Detect and handle collisions, apply damage, manage death/respawn, manage item pickup
   - Evaluate polygon-based trigger logic (activate lights, platforms, monsters)
   - Execute Lua event callbacks (pre/post-update idle hooks, entity-specific callbacks)
   - Manage level completion conditions (extermination, exploration, retrieval, repair, rescue)
4. **Replay Recording** ΓÇö If recording active, capture current action flags and world checksum delta for film
5. **Preferences & Console** ΓÇö Process console input if active; apply preference changes if requested
6. **Lua Post-Update** ΓÇö Execute Lua post-idle callbacks; update HUD script state if applicable
7. **Statistics Collection** ΓÇö Gather gameplay stats from Lua layer; queue for asynchronous upload if enabled

**Variable FPS Rendering Frame (Decoupled from Physics Tick):**
1. **Interpolated World Snapshot** ΓÇö Sample double-buffered world state at interpolation point between last two 30 FPS ticks for smooth >30 FPS rendering
2. **View Setup** (RenderMain) ΓÇö Update camera position, orientation, FOV based on interpolated player state and view effects (FOV transitions, explosion shake, teleport fold)
3. **Visibility Determination** (RenderMain) ΓÇö Ray cast from viewpoint to build hierarchical tree of visible polygons via BSP traversal
4. **Polygon Sorting** (RenderMain) ΓÇö Depth-order visible polygons with clipping window calculation
5. **Object Placement** (RenderMain) ΓÇö Project game objects into render tree with visibility culling and depth sorting
6. **Rasterization** (RenderMain) ΓÇö Traverse sorted tree, clip polygons, dispatch to backend rasterizer (software/OpenGL/shader)
7. **3D World Rendering** (Rasterizer_* / OGL_Render) ΓÇö Render textured polygons, 3D sprites, objects, models with lighting and effects
8. **HUD Rendering** (RenderOther / HUDRenderer_*) ΓÇö Render weapon/ammo/shield display, motion sensor, compass, health/oxygen bars
9. **Videos subsystem** (if recording active):
   - Main thread captures rendered frame (OpenGL FBO or SDL surface) and queues to producer buffer
   - Encode thread dequeues asynchronously, encodes to VP8, writes WebM cluster
   - Audio captured from OpenAL loopback, encoded to Vorbis, multiplexed with video
10. **Screen Composition** (RenderOther / screen.cpp) ΓÇö Composite 3D world view with HUD, terminal interface (if active), overhead map (if active), on-screen messages, fades
11. **Font & Text Rendering** (RenderOther / FontHandler) ΓÇö Render console output, dialog text, menu items via SDL or OpenGL
12. **Buffer Swap** ΓÇö Swap GL_FRONT/GL_BACK (OpenGL) or blit SDL surfaces to window framebuffer
13. **FPS Regulation** ΓÇö Sleep or yield to maintain target frame rate (typically 60+ FPS uncapped or 30 FPS V-sync)
14. **Sound** (OpenAL/SDL_mixer/OPL FM) ΓÇö Updates positional 3D audio based on entity positions; plays sound effects and music
15. **Lua scripting** (HUD, solo, stats):
    - HUD Lua executes to render overlays
    - Solo Lua executes (if single-player level defines script)
    - Stats Lua executes (if enabled)
16. **Steam parent** (if Steam build):
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
   - **Game State Serialization** ΓÇö Save current game state (player, map, entities, Lua state) if exiting mid-level
   - **Replay Finalization** ΓÇö Write film metadata and close recording file (if recording active)
   - **Preferences Flush** ΓÇö Persist modified preferences to disk (graphics, player name, key bindings, etc.)
   - **Rendering Shutdown** ΓÇö Release rasterizer resources, unload OpenGL shaders and framebuffer objects (if used), free texture and model caches, close OpenGL context
   - **Input Cleanup** ΓÇö Disable relative mouse mode, release joystick handles
   - **Resource Cleanup** ΓÇö Close resource file stack, unload shape/sound/image collections, flush file caches
   - **Logging Finalization** ΓÇö Flush and close log file
   - **Memory Deallocation** ΓÇö Free allocated world arrays, dialog surfaces, menu structures; release SDL resources
   - **Statistics Upload** ΓÇö Wait for any pending asynchronous stats POST requests to complete
   - Core engine subsystems clean up (GameWorld, RenderMain/RenderOther, Sound, Input, Files, CSeries)

2. **Steam parent** (if Steam build):
   - Wait for child process exit
   - Close bidirectional pipes
   - Call `SteamAPI_Shutdown()`

3. **Exit** ΓÇö Release SDL, close window, terminate process

---

## Data & Control Boundaries

### Entity & World State Ownership

- **GameWorld**: Owns all entity instances (player, monsters, projectiles, items, effects); other subsystems query and modify entities via GameWorld APIs, never directly allocate/deallocate
- **Interpolated World**: Double-buffered world snapshots managed by `interpolated_world.h/cpp`; rendering uses interpolated state decoupled from 30 FPS physics tick
- **Map Geometry**: Polygon/line/endpoint data immutable during gameplay; loaded atomically with level; topology validates BSP invariants at load time

### Message Ownership & Lifetimes

- **TCPMess**: `CommunicationsChannel` owns queued `Message` objects; dequeued messages are owned by `MessageDispatcher` and handler callbacks. `UninflatedMessage` (wire representation) is deallocated after deserialization. `BigChunkOfDataMessage` manages heap-allocated buffers; caller is responsible for cleanup after message handling.
- **Videos**: `Movie` singleton owns frame/audio buffers in producerΓÇôconsumer ring queue; encode thread dequeues and processes asynchronously. WebM file handle is owned by encode thread until closure.

### Resource File Stack

- Files subsystem maintains stack of open resource files with type/ID lookup; all shape/sound/image loading goes through Files subsystem to ensure consistent resource precedence (user mods override defaults)

### Action Queue Buffers

- Each networked player has a separate circular buffer in ActionQueues for storing per-frame input; game loop dequeues from local player's buffer; hub/spoke network protocol orchestrates synchronized action distribution across peers

### Rendering State

- RenderMain maintains visibility tree, polygon sort order, object placement structures; each rendering frame rebuilds these structures (not cached across frames); rasterizers are stateless consumers of these structures

### Preferences Singleton

- Preferences subsystem maintains global preference pointers accessible throughout codebase; changes persisted to disk; some preferences (key bindings, rendering backend) require re-initialization or restart to take full effect

### Lua VM Contexts

- Separate Lua VMs for embedded scripts, solo levels, netscript (multiplayer), stats, and achievements with independent persistence; scripts cannot cross-VM boundaries; state serialization per-VM on shutdown

### Audio Source Pool

- OpenALManager maintains pool of reusable OpenAL sources; priority-based allocation for new sounds; lower-priority sounds may be preempted if source pool exhausted

### Texture/Shape Collections

- Collections loaded and unloaded atomically (never partially loaded); shape collection provides immutable bitmap/animation data; loaded via Files subsystem to ensure cache consistency

### Screen Surfaces

- RenderOther maintains port-based drawing abstraction; all 2D rendering redirects through global port pointer; surfaces allocated for world view, HUD, terminal, intro, map overlay; each surface has fixed pixel format (8/16/32-bit)

### Network Topology State

- Network subsystem maintains authoritative copy of player list and game state; hub broadcasts updates to all spokes; clients maintain shadow copy for client-side prediction; mismatch detected at sync points triggers rollback

### Global State & Singletons

- **Movie** (Videos) ΓÇö Singleton managing recording state and frame queue
- **Plugins::instance()** (XML) ΓÇö Singleton for plugin discovery, validation, resource loading
- **QuickSaves** (XML) ΓÇö Singleton for enumerating and managing saved games
- **MessageInflater** (TCPMess) ΓÇö Global prototype registry for message deserialization
- **Steam parent** ΓÇö Maintains single Steamworks SDK interface pointer shared across callback handlers

### Configuration & Reset

- **XML/MML**: `ParseMMLFromFile()` populates subsystem state; `ResetAllMMLValues()` resets all subsystems to defaults. Level-specific scripts override defaults via level resource 128. Pseudo-levels (Default, Restore, End-of-game) apply common MML operations across all maps.
- **Plugin Exclusivity**: Only one HUD Lua, theme, or stats Lua allowed active. Validation occurs at load time; no runtime enforcement prevents plugin state modification mid-game.

### Process Boundary (Steam)

- **ParentΓÇôchild pipe communication**: Binary protocol encodes Steam API commands and results; messages are stateless (no transaction IDs required). Callback results are queued and flushed to child on polling intervals.

### Game State Machine

- Interface subsystem manages top-level game state (intro ΓåÆ menu ΓåÆ game ΓåÆ replay ΓåÆ epilogue); only one state active at a time; state transitions coordinated via interface callbacks (screen fading, color table swaps, resource unloading)

### Tick-Based Determinism

- Physics/AI simulation runs at fixed 30 FPS tick rate to ensure determinism across platforms and network replays; rendering decoupled via interpolation; RNG seeded at level load for reproducible sequences across platforms

---

## Notable Risks / Hotspots

- **Visibility Tree Ray-Casting Performance**: Visibility determination via ray-casting (RenderMain/RenderVisTree.cpp) iterates through all map polygons; large maps with many polygons may cause frame rate drops; no spatial acceleration structure (e.g., k-d tree) visible in subsystem descriptions
- **Entity Pool Exhaustion**: GameWorld entity lists (monsters, projectiles, items, effects, ephemera) respect dynamic limits; exceeding limits silently drops entities without warning; network mismatch possible if hub/client apply limits differently
- **Texture VRAM Budgeting**: Rasterizer tracks texture usage and purges unused textures from VRAM; no explicit budget enforcement; unbounded texture loading possible if many unique WAD collections active simultaneously
- **Audio Callback Thread Safety**: SoundManager uses lock-free queues (boost::lockfree::spsc_queue) for parameter updates between game thread and OpenAL callback thread; queue overflow silently drops updates; priority-based sound preemption may kill critical audio (e.g., player pain sounds)
- **Network Message Ordering**: UDP peer-to-peer messages may arrive out-of-order or be dropped; no explicit retransmission or sequencing in protocol (handled at TCP layer for metaserver, but raw UDP for game sync); latency/jitter may cause desynchronization
- **Lua Garbage Collection Pauses**: Lua 5.2 incremental GC may cause frame rate spikes if large Lua state accumulated (e.g., many entities with Lua behavior); no explicit GC pause budget limiting visible in subsystem descriptions
- **File I/O Blocking**: File loading (WAD, resources, images) blocks main thread; large file operations (e.g., loading 50 MB physics WAD or campaign with 100+ maps) may stall rendering
- **Color Lookup Table Consistency**: Three separate CLUTs (world_color_table, visible_color_table, interface_color_table) with gamma correction; mismatch in color palettes across rendering backends may cause visual inconsistencies
- **Platform-Specific Resource Paths**: cseries/cspaths_sdl.cpp resolves OS-specific data directories; incorrect path resolution on new OS or sandbox environment may cause resource loading failures
- **Endianness Conversion Bugs**: BStream and Packing.h provide big-endian/little-endian conversion; incorrect endianness assumptions may cause WAD parsing corruption on little-endian platforms
- **Film Playback Sync**: Recorded films stored as action flags + checksum deltas; mismatch in RNG seeding or physics constants between original recording and playback may cause divergence (desync at later frames)
- **Allocator Lifetime Issues**: Level-transition malloc allocates entity data that is freed on level exit; early exit or exception during level load may leak allocated memory; no RAII guard visible in subsystem descriptions
- **Polygon BSP Invariants**: Map geometry precalculation (map_constructors.cpp) derives side types and adjacency; invalid hand-coded map data may violate BSP invariants (e.g., back-face-polygon inconsistency), causing visibility/collision bugs
- **Scripting Sandbox Escapes**: Lua bindings expose game world mutability; no explicit sandboxing visible beyond `LuaMutabilityInterface` context checks; malicious scripts may corrupt world state or exhaust memory
- **Steam Workshop Integration**: Network/Files subsystems support Steam Workshop WAD discovery; Workshop mod conflicts or corruption may cause silent loading failures or incorrect asset precedence
- **TCPMess Message Header Framing**: 8-byte header format not explicitly specified in overviews; version mismatch or corruption could cause message deserialization failure or infinite packet reads. Timeout enforcement mitigates blocking indefinitely on malformed messages.
- **Videos ProducerΓÇôConsumer Synchronization**: Semaphore-based coordination may deadlock if frame queue fills faster than encode thread processes. No backpressure mechanism documented; main thread may stall if producer buffer is exhausted.
- **Replay Determinism (tests)**: Random seed consistency is validated against filename expectations; filename encoding format must be stable across versions. Seed extraction from gameplay assumes reproducible RNG state; any change to random number algorithm or subsystem initialization order breaks replay compatibility.
- **Steam ParentΓÇôChild Process**: Pipe communication is synchronous and blocking on message reads; long-running Steam API callbacks in parent may stall child process main loop. No documented timeout mechanism for stuck parent process.
- **QuickSave LRU Cache**: Preview image eviction on capacity limits may cause visible stutter if cache misses trigger disk I/O during gameplay. No prefetch or asynchronous loading strategy documented.
- **Plugins from Steam Workshop**: Validation occurs at discovery time; no runtime protection against malformed plugin MML or resource corruption. Resource conflicts resolved via checksums; hash collisions not addressed.
