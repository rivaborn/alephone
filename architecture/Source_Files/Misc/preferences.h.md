# Source_Files/Misc/preferences.h

## File Purpose
Declares and defines the complete preferences/configuration system for the Aleph One game engine. Manages persistent settings for graphics rendering, network gameplay, player identity, input devices, audio, and environment/map resources. All preference data is accessed globally through extern pointers to support dynamic configuration changes.

## Core Responsibilities
- Define preference data structures for all engine subsystems
- Declare initialization, loading, and persistence functions for preferences
- Manage graphics rendering options (resolution, OpenGL settings, quality levels)
- Configure network game parameters (protocol, difficulty, metaserver integration)
- Store player identity and UI preferences (name, color, team, crosshairs, chase cam)
- Define input device configuration and key/hotkey binding maps
- Track environment resources (map, physics, shapes, sounds files with checksums)
- Provide typed access to sound manager parameters and preference enums

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `graphics_preferences_data` | struct | Screen mode, OpenGL config, FPS target, movie export quality, ephemera quality |
| `network_preferences_data` | struct | Game type, difficulty, network protocol, metaserver login, port, cheat flags, UPnP settings |
| `player_preferences_data` | struct | Player name, color, team, solo profile type, difficulty, music/crosshairs toggles, camera/crosshair data |
| `input_preferences_data` | struct | Input device, modifiers (run/walk, swim/sink, auto-recenter), mouse/controller sensitivity, deadzone, key bindings |
| `environment_preferences_data` | struct | Map/physics/shapes/sounds file paths with checksums, patch data, file dialog options, Lua script settings |
| `key_binding_map` | typedef | `std::map<int, std::set<SDL_Scancode>>` for key binding storage |

Enums defined:
- `_sw_alpha_*` (software alpha blending modes)
- `_sw_driver_*` (graphics driver selection)
- `_ephemera_*` (visual quality levels: off, low, medium, high, ultra)
- `_network_game_protocol_*` (network protocol types)
- `SoloProfileType` (game variant: Aleph One, Marathon 2, Marathon Infinity)
- `_inputmod_*` (input modifier flags: run/walk swap, don't switch weapons, invert mouse, etc.)
- `_key_*` (shell key assignments for inventory, zoom, console, scores)
- `_mouse_accel_*` (mouse acceleration modes: none, classic, symmetric)

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `graphics_preferences` | `graphics_preferences_data*` | extern | Global access to graphics settings |
| `network_preferences` | `network_preferences_data*` | extern | Global access to network configuration |
| `player_preferences` | `player_preferences_data*` | extern | Global access to player profile settings |
| `input_preferences` | `input_preferences_data*` | extern | Global access to input device configuration |
| `sound_preferences` | `SoundManager::Parameters*` | extern | Global access to audio parameters |
| `environment_preferences` | `environment_preferences_data*` | extern | Global access to environment/map settings |
| `DEFAULT_MONITOR_REFRESH_FREQUENCY` | const float | file-static | Default monitor refresh rate (60 Hz) |
| `MAXIMUM_PATCHES_PER_ENVIRONMENT` | const int | file-static | Capacity for environment patches (32) |
| `NUMBER_OF_HOTKEYS` | constexpr int | file-static | Hotkey count (12) |

## Key Functions / Methods

### initialize_preferences
- Signature: `void initialize_preferences(void)`
- Purpose: Initialize the preferences system on engine startup; allocate and populate all preference structures
- Inputs: None
- Outputs/Return: None (modifies global extern pointers)
- Side effects: Allocates memory for all preference structures; may read from defaults
- Calls: Not visible in this file
- Notes: Must be called before any preference access

### read_preferences
- Signature: `void read_preferences()`
- Purpose: Load user preferences from persistent storage
- Inputs: None
- Outputs/Return: None (populates extern preference pointers)
- Side effects: File I/O; reads preferences file from disk
- Calls: Not visible
- Notes: Called during initialization or when user requests reload

### write_preferences
- Signature: `void write_preferences()`
- Purpose: Save all current preferences to persistent storage
- Inputs: None
- Outputs/Return: None
- Side effects: File I/O; writes all preference structures to disk
- Calls: Not visible
- Notes: Called on shutdown or after user modifies settings

### handle_preferences
- Signature: `void handle_preferences(void)`
- Purpose: Process preference changes via UI dialogs and apply them to running engine
- Inputs: None
- Outputs/Return: None
- Side effects: May invoke dialogs; updates global preference data
- Calls: Not visible
- Notes: Bridge between UI layer and preference system

### transition_preferences
- Signature: `void transition_preferences(const DirectorySpecifier& legacy_prefs_dir)`
- Purpose: Migrate preferences from legacy directory structure to current format
- Inputs: `legacy_prefs_dir` ΓÇö path to old preferences directory
- Outputs/Return: None
- Side effects: File I/O; reads legacy files and converts/writes to new format
- Calls: Not visible
- Notes: Compatibility function for version upgrades

### get_fps_target
- Signature: `static inline int16 get_fps_target()`
- Purpose: Accessor for target framerate setting
- Inputs: None
- Outputs/Return: `int16` fps target (0 = unlimited, else multiple of 30)
- Side effects: None (read-only)
- Calls: Accesses `graphics_preferences->fps_target`
- Notes: Trivial inline accessor; used during frame pacing logic

## Control Flow Notes

This file defines the **configuration layer** for the engine, accessed during:
- **Initialization**: `initialize_preferences()` called early in `initialize_application()` (from shell.h)
- **Load phase**: `read_preferences()` restores user settings from disk
- **Frame loop**: Preference values (FPS target, graphics flags, input modifiers) consulted continuously
- **Configuration phase**: `handle_preferences()` processes user UI changes; `write_preferences()` persists them
- **Shutdown**: Preferences saved before exit
- **Upgrade**: `transition_preferences()` migrates old settings to new version format

Chase cam and crosshair data flow through player dialogs (Configure_ChaseCam, Configure_Crosshairs in ChaseCam.h/Crosshairs.h) ΓåÆ stored in `player_preferences_data` ΓåÆ applied at render time.

## External Dependencies

- **Includes**: 
  - `interface.h` ΓÇö game interface types and shape descriptor macros
  - `ChaseCam.h` ΓÇö ChaseCamData struct and chase cam interface
  - `Crosshairs.h` ΓÇö CrosshairData struct and crosshair interface
  - `OGL_Setup.h` ΓÇö OGL_ConfigureData struct for OpenGL settings
  - `shell.h` ΓÇö screen_mode_data, BobbingType, PREFERENCES_NAME_LENGTH constant
  - `SoundManager.h` ΓÇö SoundManager::Parameters nested class
  - `<map>`, `<set>` ΓÇö STL containers for key bindings

- **Types defined elsewhere and used**:
  - `SDL_Scancode` ΓÇö from SDL library
  - `_fixed` ΓÇö fixed-point numeric type (likely from cseries.h)
  - `TimeType` ΓÇö timestamp type (likely from world.h)
  - `FilmProfileType` ΓÇö film format enum (not defined in this file)
  - `rgb_color` / `RGBColor` ΓÇö color type (from cseries.h or interface.h)
  - `DirectorySpecifier` ΓÇö file path abstraction (forward-declared, defined elsewhere)
  - `angle` ΓÇö angle type from game geometry (from world.h)
