# Source_Files/Sound/OpenALManager.cpp

## File Purpose
Implements OpenAL audio device management and playback orchestration for the Aleph One game engine. Handles device initialization, audio player lifecycle, source pooling, 3D listener updates, and SDL integration for audio mixing.

## Core Responsibilities
- OpenAL device, context, and source initialization
- Audio player (sound/music/stream) lifecycle and scheduling
- Audio source pooling with priority-based allocation
- 3D spatial audio listener updates and coordinate conversion
- Master/music volume synchronization across active players
- Audio queue processing and SDL mixer callback integration

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `AudioParameters` | struct | Audio config: sample rate, channels, frame size, HRTF, 3D enable, master/music volume |
| `OptionalExtension` | enum | Supported OpenAL extensions: Spatialization, DirectChannelRemix |
| `HrtfSupport` | enum | HRTF capability status: Supported, Unsupported, Required |
| `AudioPlayer::AudioSource` | struct | Wraps OpenAL source ID and multi-buffer pool |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `instance` | OpenALManager\* | static | Singleton manager instance |
| `alcLoopbackOpenDeviceSOFT`, `alcIsRenderFormatSupportedSOFT`, `alcRenderSamplesSOFT`, `alGetStringiSOFT`, `alGenFilters`, `alDeleteFilters`, `alFilteri`, `alFilterf` | Function pointers (LPXXX) | static | OpenAL extension functions, dynamically loaded |
| `p_ALCDevice` | ALCdevice\* | instance | OpenAL loopback audio device |
| `p_ALCContext` | ALCcontext\* | instance | OpenAL context for current device |
| `master_volume`, `music_volume` | std::atomic\<float\> | instance | Playback volume levels (thread-safe) |
| `listener_location` | AtomicStructure\<world_location3d\> | instance | 3D listener position (thread-safe) |
| `sources_pool` | std::queue\<AudioSource\> | instance | Pre-allocated unused OpenAL sources |
| `audio_players_queue` | std::deque\<shared_ptr\<AudioPlayer\>\> | instance | Active players (audio thread only, no locking) |
| `audio_players_shared` | boost::lockfree::spsc_queue\<shared_ptr\<AudioPlayer\>\> | instance | MainΓåÆaudio thread pipeline |
| `low_pass_filter` | ALuint | instance | Low-pass filter object ID |

## Key Functions / Methods

### Init (static)
- **Signature:** `static bool Init(const AudioParameters& parameters)`
- **Purpose:** Initialize OpenALManager singleton; load extension functions, create device/context, allocate sources/effects.
- **Inputs:** `parameters` ΓÇô audio config (rate, channels, HRTF, etc.)
- **Outputs/Return:** `true` if device, sources, and effects initialized successfully
- **Side effects:** Creates static OpenALManager instance; loads extension function pointers; allocates OpenAL resources
- **Calls:** `Shutdown()`, `OpenDevice()`, `LoadOptionalExtensions()`, `GenerateSources()`, `GenerateEffects()`
- **Notes:** Reuses existing instance if parameters match; shuts down and recreates if parameters differ. Extension functions only loaded once on first Init.

### ProcessAudioQueue
- **Signature:** `void ProcessAudioQueue()`
- **Purpose:** Transfer pending players from lockfree shared queue to work queue; update listener; iterate players calling Update/Play; recycle stopped players' sources.
- **Inputs:** None (reads from `audio_players_shared`)
- **Outputs/Return:** None
- **Side effects:** Modifies `audio_players_queue` and `sources_pool`; calls Update/Play on all players; updates OpenAL listener state
- **Calls:** `UpdateListener()`, `AudioPlayer::AssignSource()`, `AudioPlayer::Update()`, `AudioPlayer::Play()`, `RetrieveSource()`
- **Notes:** Called from SDL mixer callback; checks `stop_signal` and return value of Update/Play to determine if player continues. Round-robin scheduling via push-to-back.

### UpdateListener
- **Signature:** `void UpdateListener()`
- **Purpose:** Read listener 3D position and convert to OpenAL world coords; update AL_POSITION, AL_ORIENTATION, AL_VELOCITY.
- **Inputs:** None (reads from `listener_location`)
- **Outputs/Return:** None
- **Side effects:** Calls alListenerfv three times; early return if `!sounds_3d`
- **Calls:** `listener_location.Get()`, `alListenerfv()` (via OpenAL C API)
- **Notes:** Converts game yaw/pitch angles via `angleConvert` and `degreToRadian`; swaps YΓåöZ to convert from game coords to OpenGL/OpenAL coords; scales position/velocity by `WORLD_ONE`.

### PlaySound / PlayMusic / PlayStream
- **Signature:** 
  - `std::shared_ptr<SoundPlayer> PlaySound(const Sound& sound, const SoundParameters& parameters)`
  - `std::shared_ptr<MusicPlayer> PlayMusic(std::vector<MusicPlayer::Sequence>& sequences, uint32_t starting_sequence_index, uint32_t starting_segment_index, const MusicParameters& parameters)`
  - `std::shared_ptr<StreamPlayer> PlayStream(CallBackStreamPlayer callback, uint32_t rate, bool stereo, AudioFormat audioFormat, void* userdata)`
- **Purpose:** Factory methods to create and enqueue audio players for playback.
- **Inputs:** Sound/music/stream parameters specific to player type
- **Outputs/Return:** Shared pointer to created player (or empty ptr if not active)
- **Side effects:** Creates player object, pushes to `audio_players_shared` lockfree queue
- **Calls:** Player constructors, `audio_players_shared.push()`
- **Notes:** Early return if `!process_audio_active`; callers hold reference to control playback.

### PickAvailableSource
- **Signature:** `std::unique_ptr<AudioPlayer::AudioSource> PickAvailableSource(const AudioPlayer& audioPlayer)`
- **Purpose:** Allocate a source for a player: return from pool if available, else steal from lowest-priority player.
- **Inputs:** `audioPlayer` ΓÇô requesting player (to compare priority)
- **Outputs/Return:** Unique pointer to AudioSource, or nullptr if no source available
- **Side effects:** Pops from `sources_pool` or calls `RetrieveSource()` on victim player
- **Calls:** `audio_players_queue` iteration, `AudioPlayer::GetPriority()`, `AudioPlayer::RetrieveSource()`
- **Notes:** Uses `std::min_element` to find lowest-priority player; only steals if victim priority < requester priority.

### GenerateSources
- **Signature:** `bool GenerateSources()`
- **Purpose:** Query device for available mono/stereo sources; allocate ALuint sources; wrap each in AudioSource with buffer pool; populate `sources_pool`.
- **Inputs:** None (reads `p_ALCDevice`, `audio_parameters`)
- **Outputs/Return:** `true` if allocation succeeded
- **Side effects:** Allocates OpenAL sources and buffers; populates `sources_pool`; logs errors on failure
- **Calls:** `alGenSources()`, `alSourcei()`, `alSourceRewind()`, `alGenBuffers()`, `alGetError()`
- **Notes:** Queries ALC_MONO_SOURCES and ALC_STEREO_SOURCES; creates `num_buffers` per source for streaming. Returns false and logs if any OpenAL call fails.

### OpenDevice
- **Signature:** `bool OpenDevice()`
- **Purpose:** Create OpenAL loopback device and context with desired audio format and parameters.
- **Inputs:** None (reads `audio_parameters`)
- **Outputs/Return:** `true` if device and context created and made current
- **Side effects:** Allocates `p_ALCDevice` and `p_ALCContext`; sets OpenAL context as current; logs errors on failure
- **Calls:** `alcLoopbackOpenDeviceSOFT()`, `GetBestOpenALSupportedFormat()`, `alcCreateContext()`, `alcMakeContextCurrent()`
- **Notes:** Uses context attributes (format, channels, frequency, HRTF) from `audio_parameters`. Early return if device already open.

### CloseDevice
- **Signature:** `bool CloseDevice()`
- **Purpose:** Deactivate and release OpenAL device and context.
- **Inputs:** None
- **Outputs/Return:** `true` if cleanup succeeded
- **Side effects:** Nullifies `p_ALCDevice` and `p_ALCContext`; logs errors on failure
- **Calls:** `alcMakeContextCurrent()`, `alcDestroyContext()`, `alcCloseDevice()`
- **Notes:** Calls in reverse order: unset current context, destroy context, close device.

### GenerateEffects
- **Signature:** `bool GenerateEffects()`
- **Purpose:** Create low-pass filter object and initialize gain parameters.
- **Inputs:** None
- **Outputs/Return:** `true` if filter creation succeeded
- **Side effects:** Allocates `low_pass_filter` and sets AL_FILTER_TYPE, AL_LOWPASS_GAIN, AL_LOWPASS_GAINHF
- **Calls:** `alGenFilters()`, `alFilteri()`, `alFilterf()`, `alGetError()`

### GetPlayBackAudio
- **Signature:** `void GetPlayBackAudio(uint8* data, int length)`
- **Purpose:** Process audio queue and render mixed samples into buffer (for recording loopback device).
- **Inputs:** `data` ΓÇô output buffer, `length` ΓÇô buffer size in bytes
- **Outputs/Return:** None
- **Side effects:** Calls ProcessAudioQueue; renders samples via `alcRenderSamplesSOFT`
- **Calls:** `ProcessAudioQueue()`, `alcRenderSamplesSOFT()`

### GetHrtfSupport
- **Signature:** `HrtfSupport GetHrtfSupport() const`
- **Purpose:** Query device HRTF capability.
- **Outputs/Return:** HrtfSupport enum (Supported, Unsupported, or Required)
- **Calls:** `alcGetIntegerv(p_ALCDevice, ALC_HRTF_STATUS_SOFT, ...)`

### MixerCallback (static)
- **Signature:** `static void MixerCallback(void* usr, uint8* stream, int len)`
- **Purpose:** SDL audio callback; compute frame size and dispatch to GetPlayBackAudio.
- **Inputs:** `usr` ΓÇô OpenALManager instance pointer (opaque to SDL), `stream` ΓÇô audio buffer, `len` ΓÇô buffer size in bytes
- **Outputs/Return:** None
- **Side effects:** Calls GetPlayBackAudio
- **Calls:** `GetPlayBackAudio()`
- **Notes:** Casts `usr` to OpenALManager\*; computes frame size from SDL spec (channels ├ù bit depth / 8).

### UpdateParameters
- **Signature:** `void UpdateParameters(const AudioParameters& parameters)`
- **Purpose:** Store audio parameters and apply volume settings.
- **Inputs:** `parameters` ΓÇô new audio config
- **Outputs/Return:** None
- **Side effects:** Updates `audio_parameters`, `master_volume`, `music_volume`
- **Calls:** `SetMasterVolume()`, `SetMusicVolume()`

**Other helper methods (brief):**
- `SetMasterVolume()` / `SetMusicVolume()` ΓÇô clamp to range [0,1] / [0,10], update atomic, call `ResyncPlayers()`
- `ResyncPlayers()` ΓÇô mark players out-of-sync with OpenAL params (forces volume re-apply)
- `RetrieveSource()` ΓÇô extract source from player, return to `sources_pool`
- `StopAllPlayers()` ΓÇô clear queues, mark all players inactive
- `LoadOptionalExtensions()` ΓÇô query support for AL_SOFT_source_spatialize and AL_SOFT_direct_channels_remix
- `CleanEverything()` ΓÇô deallocate sources, buffers, filter, close device
- `GetBestOpenALSupportedFormat()` ΓÇô iterate format_type list to find device-supported format

## Control Flow Notes
**Initialization (one-time):**
1. Engine calls `Init(params)` ΓåÆ loads extensions, creates singleton, opens device, allocates sources/effects
2. Constructor calls `SDL_OpenAudio(desired, &obtained)` with `MixerCallback`
3. SDL audio thread starts, periodically calls `MixerCallback`

**Per-frame playback (main thread):**
1. Game calls `PlaySound()`, `PlayMusic()`, or `PlayStream()` ΓåÆ enqueue to `audio_players_shared`
2. Main thread may call `SetMasterVolume()`, `UpdateListener()`, `Pause()`, `Stop()`, etc.

**SDL audio callback (audio thread):**
1. SDL calls `MixerCallback` ΓåÆ computes frame size ΓåÆ calls `GetPlayBackAudio`
2. `GetPlayBackAudio` ΓåÆ `ProcessAudioQueue` ΓåÆ pulls from shared queue, updates/plays each player, recycles sources
3. OpenAL renders mixed audio to output buffer

**Shutdown:**
1. Engine calls `Shutdown()` ΓåÆ deletes singleton
2. Destructor calls `CleanEverything()` ΓåÆ deallocates sources, buffers, filter; closes device; calls `SDL_CloseAudio()`

## External Dependencies
- **OpenAL-Soft:** alcLoopbackOpenDeviceSOFT, alcCreateContext, alcMakeContextCurrent, alGenSources, alListenerfv, alGenFilters, etc.
- **SDL:** SDL_OpenAudio, SDL_PauseAudio, SDL_GetAudioStatus, SDL_LockAudio, SDL_UnlockAudio, SDL_CloseAudio
- **Classes (defined elsewhere):** SoundPlayer, MusicPlayer, StreamPlayer, AudioPlayer
- **boost::lockfree:** spsc_queue (single-producer, single-consumer)
- **Logging.h:** logError macro
