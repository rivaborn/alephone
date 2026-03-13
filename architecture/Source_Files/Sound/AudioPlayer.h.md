# Source_Files/Sound/AudioPlayer.h

## File Purpose
Defines the `AudioPlayer` abstract base class for managing OpenAL audio playback in the Aleph One game engine. Coordinates audio buffer management, OpenAL source configuration, and integrates with the OpenALManager for resource allocation and prioritization. Provides lock-free synchronization for real-time audio parameter updates.

## Core Responsibilities
- Abstract interface for audio playback with OpenAL sources and buffers
- Manage buffer queue lifecycle (fill ΓåÆ queue ΓåÆ unqueue ΓåÆ refill)
- Convert audio format specifications to OpenAL format enums
- Support dynamic parameter updates (gain, pitch, position) via lock-free double-buffering
- Handle stop/rewind control signals
- Provide priority-based scheduling for OpenALManager

## Key Types / Data Structures
| Name | Kind | Purpose |
| --- | --- | --- |
| SetupALResult | typedef | Pair<bool, bool>: config success and full setup completion status |
| AtomicStructure<T> | template class | Lock-free double-buffering with SPSC queue for thread-safe updates |
| AudioSource | struct | Container holding OpenAL source ID and buffer map |
| AudioPlayerBuffers | typedef | Unordered map of buffer ID ΓåÆ queued-for-processing status |
| mapping_audio_format_openal | static const map | Maps (AudioFormat, stereo) pairs to OpenAL format enums |

## Global / File-Static State
| Name | Type | Scope | Purpose |
| --- | --- | --- | --- |
| num_buffers | constexpr uint32_t | static | Audio buffer pool size (4 buffers) |
| buffer_samples | constexpr uint32_t | static | Samples per buffer (8192) |
| mapping_audio_format_openal | static inline const boost::unordered_map | static | Format conversion table to OpenAL constants |

## Key Functions / Methods

### Update
- Signature: `bool Update()`
- Purpose: Main per-frame processingΓÇöunqueue finished buffers, fill empty buffers, update AL source parameters
- Inputs: none
- Outputs: `bool` (likely playback status)
- Side effects: Modifies OpenAL source state and buffer queue
- Calls: `UnqueueBuffers()`, `FillBuffers()`, `SetUpALSourceIdle()`
- Notes: Called by OpenALManager via friend relationship; core frame loop entry point

### SetUpALSourceInit
- Signature: `virtual bool SetUpALSourceInit()`
- Purpose: One-time OpenAL source parameter initialization (gain, pitch, position, velocity)
- Inputs: none
- Outputs: `bool` (success)
- Side effects: Configures OpenAL source
- Calls: OpenAL AL* functions
- Notes: Overridable by subclasses; called once at source assignment

### SetUpALSourceIdle
- Signature: `virtual SetupALResult SetUpALSourceIdle()`
- Purpose: Per-frame OpenAL parameter updates; detects incomplete configuration
- Inputs: none
- Outputs: `SetupALResult` (config success, fully setup)
- Side effects: May update AL source state; uses locks per comment
- Calls: OpenAL functions
- Notes: Overridable; called every frame for dynamic parameter changes

### GetNextData (pure virtual)
- Signature: `virtual uint32_t GetNextData(uint8* data, uint32_t length) = 0`
- Purpose: Subclass-provided audio data source; fills buffer with next audio chunk
- Inputs: `data` (output buffer), `length` (max bytes)
- Outputs: `uint32_t` (bytes written)
- Side effects: Fills provided buffer
- Calls: (subclass-specific)
- Notes: Must be implemented by concrete subclasses

### Public Control/Query Methods
- `AskStop()`: Signal playback stop
- `IsActive()`: Atomic query of active state
- `AskRewind()`: Signal rewind request
- `GetPriority()`: Pure virtual; return priority for scheduling
- `IsPlaying()`: Check if source is actively playing
- `HasBufferFormatChanged()`: Detect format changes requiring rebuffering
- `LoadParametersUpdates()`: Virtual; load queued parameter changes (default no-op)

## Control Flow Notes
Typical lifecycle: **Init** ΓåÆ **AssignSource** ΓåÆ **[Update ΓåÆ UnqueueBuffers ΓåÆ FillBuffers ΓåÆ SetUpALSourceIdle]** (per frame) ΓåÆ **ResetSource/Rewind** (on signals). The `AtomicStructure` template and lock-free queue enable parameter updates without blocking the audio thread. OpenALManager acts as the external coordinator via friend access.

## External Dependencies
- **OpenAL**: `<AL/al.h>`, `<AL/alext.h>`ΓÇöaudio backend (defines `ALuint`, `AL_FORMAT_*` constants)
- **Boost**: `lockfree::spsc_queue`, `unordered_map`ΓÇölock-free concurrency and hashing
- **Local**: `Decoder.h`ΓÇöaudio stream decoding interface; `AudioFormat` enum (defined elsewhere)
- **Standard**: `<atomic>`, `<algorithm>`, `<unordered_map>`, `<memory>` (std::unique_ptr)
