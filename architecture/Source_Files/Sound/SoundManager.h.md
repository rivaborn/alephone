# Source_Files/Sound/SoundManager.h

## File Purpose

Central singleton manager for all audio playback in the Aleph One engine. Handles sound loading, playback, volume/pitch control, spatial audio positioning, and ambient sound management. Coordinates between the sound file system and individual SoundPlayer instances.

## Core Responsibilities

- Initialize and shutdown the audio subsystem with configurable parameters (sample rate, buffer size, master volume)
- Load/unload sounds from disk and manage in-memory sound caches
- Create and manage SoundPlayer instances for active sound playback
- Compute spatial audio parameters (stereo panning, attenuation based on listener/source position)
- Track and update listener location and orientation for 3D audio
- Manage ambient sound sources separately with dedicated channels and buffer allocation
- Provide pause/resume control via RAII Pause class
- Convert sound indices (e.g., random sound permutations) to actual playable sound indices
- Calculate pitch modifiers and obstruction effects (muffling/attenuation)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `SoundManager::Parameters` | struct | Configuration for audio subsystem (volume_db, sample rate, channels, music volume) |
| `SoundManager::SoundVolumes` | struct | Stereo/mono volume breakdown (volume, left_volume, right_volume) |
| `SoundManager::Pause` | class | RAII guard to pause/resume all sounds on scope entry/exit |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_instance` | `SoundManager*` | static (singleton) | Single global sound manager instance |
| `DEFAULT_SOUND_LEVEL_DB` | `float` | static const | Default master volume in dB (-8.0) |
| `MAXIMUM_VOLUME_DB` | `float` | static const | Maximum allowed volume (0 dB) |
| `MINIMUM_VOLUME_DB` | `float` | static const | Minimum allowed volume (-40 dB) |
| `DEFAULT_MUSIC_LEVEL_DB` | `float` | static const | Default music volume in dB (-12.0) |
| `MAX_SOUNDS_FOR_SOURCE` | `int` | static const | Max concurrent sounds per source (3) |
| `MAXIMUM_AMBIENT_SOUND_CHANNELS` | `int` | static const | Max concurrent ambient sound channels (4) |
| `MAXIMUM_SOUND_BUFFER_SIZE` | `int` | static const | Max audio buffer allocation (1 MB) |
| `MINIMUM_SOUND_PITCH` | `int` | static const | Min pitch shift (1) |
| `MAXIMUM_SOUND_PITCH` | `int` | static const | Max pitch shift (256 * FIXED_ONE) |

## Key Functions / Methods

### instance()
- **Signature:** `static inline SoundManager* instance()`
- **Purpose:** Accessor for singleton instance; creates on first call
- **Outputs/Return:** Pointer to global SoundManager
- **Side effects:** Allocates SoundManager on heap if first call
- **Notes:** Thread-unsafe; assumes single-threaded initialization

### Initialize(Parameters&)
- **Signature:** `void Initialize(const Parameters&)`
- **Purpose:** Configure and start the audio system
- **Inputs:** Parameters struct with sample rate, buffer size, volume levels, channel configuration
- **Side effects:** Allocates SoundFile, SoundMemoryManager, sets `initialized` and `active` flags

### PlaySound() overloads
- **Signature (1):** `std::shared_ptr<SoundPlayer> PlaySound(LoadedResource& rsrc, const SoundParameters& parameters)`
- **Signature (2):** `std::shared_ptr<SoundPlayer> PlaySound(short sound_index, world_location3d *source, short identifier, _fixed pitch, bool soft_rewind)`
- **Signature (3):** `std::shared_ptr<SoundPlayer> DirectPlaySound(short sound_index, angle direction, short volume, _fixed pitch)`
- **Purpose:** Create and start playback of a sound; overloads support raw resource, indexed sound with spatial positioning, or direct playback
- **Inputs:** Sound index/resource, optional source location, pitch modifier, volume (in Signature 3: direction angle)
- **Outputs/Return:** Shared pointer to active SoundPlayer
- **Side effects:** Allocates SoundPlayer, calls `BufferSound()`, updates `sound_players` set
- **Calls:** `BufferSound()`, `CalculateSoundVariables()`, `GetSoundDefinition()`, `ManageSound()`
- **Notes:** Spatial sounds require source location; DirectPlaySound skips spatial calculation

### StopSound(short, short) / StopAllSounds()
- **Signature:** `void StopSound(short identifier, short sound_index)` and `void StopAllSounds()`
- **Purpose:** Halt playback of a specific sound or all active sounds
- **Inputs:** Sound identifier and source identifier (for StopSound)
- **Side effects:** Removes SoundPlayer from active tracking sets
- **Calls:** `GetSoundPlayer()` (for StopSound)

### UpdateListener()
- **Signature:** `void UpdateListener()`
- **Purpose:** Update spatial audio listener position (called per frame)
- **Side effects:** Recomputes stereo panning and attenuation for all active sounds based on current listener location
- **Calls:** `_sound_listener_proc()` (external callback), `CalculateSoundVariables()` for all active players

### LoadSound(short) / UnloadSound(short)
- **Signature:** `bool LoadSound(short sound)` and `void UnloadSound(short sound)`
- **Purpose:** Pre-load/unload a sound into memory via SoundMemoryManager
- **Inputs:** Sound index
- **Outputs/Return:** Success flag (LoadSound only)

### Idle()
- **Signature:** `void Idle()`
- **Purpose:** Per-frame audio update; clean up finished sound players
- **Side effects:** Calls `ManagePlayers()`, updates transitions, removes inactive SoundPlayers
- **Calls:** `ManagePlayers()`, `CleanInactivePlayers()`

### AdjustVolumeUp(short) / AdjustVolumeDown(short)
- **Signature:** `bool AdjustVolumeUp(short sound_index)` and `bool AdjustVolumeDown(short sound_index)`
- **Purpose:** Increment/decrement master or per-sound volume
- **Inputs:** Optional sound index (NONE for master volume)
- **Outputs/Return:** Success flag

### CauseAmbientSoundSourceUpdate()
- **Signature:** `void CauseAmbientSoundSourceUpdate()`
- **Purpose:** Trigger re-evaluation of all ambient sound sources
- **Side effects:** Calls `UpdateAmbientSoundSources()`, manages `ambient_sound_players` set

### From_db(float, bool)
- **Signature:** `static float From_db(float db, bool music)`
- **Purpose:** Convert dB to linear gain (inverse of dB logarithm formula)
- **Inputs:** Volume in dB, flag for music vs. SFX threshold
- **Outputs/Return:** Linear gain multiplier (0.0ΓÇô1.0+)
- **Notes:** Returns 0 if below minimum threshold; formula is `10^(db/20)`

### GetCurrentAudioTick()
- **Signature:** `static uint64_t GetCurrentAudioTick()`
- **Purpose:** Retrieve current playback position in audio frames/ticks
- **Outputs/Return:** Tick count

## Control Flow Notes

**Initialization**: `Initialize()` is called at engine startup with audio parameters. Sets `initialized` and `active` flags, allocates SoundFile and SoundMemoryManager.

**Per-frame**: `Idle()` is called each game frame to:
- Manage active SoundPlayer instances
- Clean up finished players
- Update listener position and recompute spatial audio
- Handle ambient sound updates

**Shutdown**: `Shutdown()` closes SoundFile and releases resources.

**Pause**: The `Pause` RAII class calls `SetStatus(false)` on construction and `SetStatus(true)` on destruction, pausing and resuming all sound playback without deallocating players.

## External Dependencies

- **Includes:**
  - `cseries.h` ΓÇö platform macros, data types, SDL headers
  - `FileHandler.h` ΓÇö FileSpecifier, OpenedFile, LoadedResource
  - `SoundFile.h` ΓÇö SoundFile, SoundDefinition, SoundHeader
  - `world.h` ΓÇö world_location3d, angle, world_distance
  - `SoundPlayer.h` ΓÇö SoundPlayer class and SoundParameters
  - `<set>` ΓÇö std::set for active player tracking

- **External symbols (defined elsewhere):**
  - `world_location3d *_sound_listener_proc()` ΓÇö callback providing listener location/orientation
  - `uint16 _sound_obstructed_proc(world_location3d*, bool)` ΓÇö callback checking sound obstruction
  - `void _sound_add_ambient_sources_proc(void*, add_ambient_sound_source_proc_ptr)` ΓÇö callback for ambient source enumeration
  - `SoundMemoryManager` ΓÇö forward declaration, manages sound cache
  - Various sound accessor functions: `Sound_TerminalLogon()`, `Sound_ButtonSuccess()`, etc.
  - `parse_mml_sounds()`, `reset_mml_sounds()` ΓÇö MML sound definition parsing
