# Source_Files/Sound/OpenALManager.h

## File Purpose
Central audio management system for an OpenAL-based game engine. Initializes and maintains OpenAL device/context, manages playback of sounds/music/streams, handles listener positioning for 3D audio, and maps between OpenAL and SDL audio formats.

## Core Responsibilities
- Initialize and shutdown OpenAL device and audio context
- Manage lifecycle of audio players (SoundPlayer, MusicPlayer, StreamPlayer)
- Handle pause/resume/stop operations for all active audio
- Update listener position and retrieve audio parameters (volume, frequency)
- Provide source pooling and allocation for audio playback
- Map and negotiate audio format/channel configurations between OpenAL and SDL
- Load and manage optional OpenAL extensions (spatialization, direct channel remix)
- Support loopback device for audio capture/mixing
- Support HRTF (Head-Related Transfer Function) for spatial audio

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `AudioParameters` | struct | Audio configuration (sample rate, channels, volume levels, HRTF, 3D sound flags) |
| `OptionalExtension` | enum class | OpenAL optional feature flags (Spatialization, DirectChannelRemix) |
| `HrtfSupport` | enum class | HRTF capability levels (Supported, Unsupported, Required) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `instance` | `OpenALManager*` | static | Singleton instance for global access |
| `alcLoopbackOpenDeviceSOFT`, `alcIsRenderFormatSupportedSOFT`, `alcRenderSamplesSOFT` | function pointers | static | Loopback device extension functions |
| `alGetStringiSOFT` | function pointer | static | String enumeration extension function |
| `alGenFilters`, `alDeleteFilters`, `alFilteri`, `alFilterf` | function pointers | static | Audio filter object functions |

## Key Functions / Methods

### Init
- Signature: `static bool Init(const AudioParameters& parameters)`
- Purpose: Initialize OpenAL device, context, and audio system; set up source pools and effects
- Inputs: AudioParameters (rate, channels, volumes, HRTF mode, 3D sound flag)
- Outputs/Return: bool (success/failure)
- Side effects: Creates singleton instance, opens OpenAL device, loads extensions, starts audio processing

### Shutdown
- Signature: `static void Shutdown()`
- Purpose: Clean up OpenAL resources and stop audio processing
- Inputs: None
- Outputs/Return: void
- Side effects: Stops all players, closes device/context, deallocates sources

### PlaySound / PlayMusic / PlayStream
- Signature: `std::shared_ptr<SoundPlayer> PlaySound(const Sound&, const SoundParameters&)` (and similar for Music/Stream)
- Purpose: Create and queue an audio player for playback
- Inputs: Audio data/parameters and playback parameters
- Outputs/Return: Shared pointer to active player
- Side effects: Adds player to audio queue, allocates audio source from pool

### PickAvailableSource
- Signature: `std::unique_ptr<AudioPlayer::AudioSource> PickAvailableSource(const AudioPlayer&)`
- Purpose: Retrieve an available audio source from the pool for playback
- Inputs: Reference audio player (for priority context)
- Outputs/Return: Unique pointer to AudioSource
- Side effects: Removes source from pool; may stop lower-priority players if pool exhausted

### UpdateListener
- Signature: `void UpdateListener(world_location3d listener)`
- Purpose: Update listener position for 3D audio spatial calculations
- Inputs: Listener world location
- Outputs/Return: void
- Side effects: Updates atomic listener_location (thread-safe)

### SetMasterVolume / SetMusicVolume
- Signature: `void SetMasterVolume(float volume)` / `void SetMusicVolume(float volume)`
- Purpose: Update volume levels
- Inputs: Float volume (0.0ΓÇô1.0 range)
- Outputs/Return: void
- Side effects: Updates atomic volume members; ResyncPlayers called to propagate changes

### Pause / Start / Stop
- Signature: `void Pause(bool paused)`, `void Start()`, `void Stop()`
- Purpose: Control global audio playback state
- Inputs: Boolean flag (for Pause) or none
- Outputs/Return: void
- Side effects: Sets paused_audio flag, may pause all active players or call ProcessAudioQueue

### GetHrtfSupport / IsHrtfEnabled / IsExtensionSupported
- Signature: `HrtfSupport GetHrtfSupport() const`, `bool IsHrtfEnabled() const`, `bool IsExtensionSupported(OptionalExtension) const`
- Purpose: Query HRTF and extension capabilities
- Inputs: OptionalExtension enum (for last method)
- Outputs/Return: HrtfSupport enum or bool
- Side effects: None (query only)

### GetPlayBackAudio
- Signature: `void GetPlayBackAudio(uint8* data, int length)`
- Purpose: Fill audio buffer for loopback device (audio capture)
- Inputs: Pointer to output buffer, length in bytes
- Outputs/Return: void
- Side effects: Renders audio samples from all active players into buffer

## Control Flow Notes
OpenALManager follows a typical game audio system lifecycle: **Init** (once at startup) ΓåÆ **PlaySound/Music/Stream** (on demand during gameplay) ΓåÆ **UpdateListener** (per frame for 3D positioning) ΓåÆ **Pause/Resume** (on game state change) ΓåÆ **Shutdown** (at exit). Audio processing occurs asynchronously via ProcessAudioQueue (likely called from dedicated audio thread). MixerCallback bridges SDL audio callbacks to internal player queues using lock-free SPSC/MPSC queues.

## External Dependencies
- **OpenAL Soft**: ALCdevice, ALCcontext, AL* filter functions, ALC format/channel enums
- **SDL2**: SDL_AudioSpec, SDL_AudioFormat, SDL_AudioCallback
- **Boost**: boost::lockfree::spsc_queue (lock-free single-producer single-consumer)
- **Custom headers**: MusicPlayer.h, SoundPlayer.h, StreamPlayer.h (audio player implementations); implied AudioPlayer base class, StreamDecoder, world_location3d, AtomicStructure utility
- **Math**: M_PI, std::math constants for angle conversion
