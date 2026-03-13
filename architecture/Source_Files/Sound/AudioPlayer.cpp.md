# Source_Files/Sound/AudioPlayer.cpp

## File Purpose
Implementation of `AudioPlayer`, a base class for audio playback in the Aleph One engine. Manages OpenAL source lifecycle, buffer queue operations, format tracking, and playback control.

## Core Responsibilities
- Allocate, reset, and retrieve OpenAL audio sources via `OpenALManager`
- Manage a queue of 4 buffers (8192 samples each) for streamed audio
- Track and validate audio format (sample rate, mono/stereo, bit depth)
- Implement playback flow: fill buffers ΓåÆ queue ΓåÆ play ΓåÆ unqueue processed buffers
- Synchronize audio parameters with OpenAL when properties change
- Handle rewind/stop signals via atomic flags
- Support subclasses by providing virtual hooks for data generation and parameter updates

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `AudioSource` | struct (private) | Holds OpenAL source ID and buffer usage map |
| `AudioPlayerBuffers` | typedef | Map of buffer ID ΓåÆ queued status |
| `SetupALResult` | typedef | Pair<success, full_sync> for AL parameter updates |
| `AtomicStructure<T>` | template (header) | Lock-free double-buffering for cross-thread updates |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `num_buffers` | constexpr uint32_t (4) | header-static | Queue depth for streaming |
| `buffer_samples` | constexpr uint32_t (8192) | header-static | Samples per buffer |
| `mapping_audio_format_openal` | static inline map | header-static | Maps (AudioFormat, stereo) ΓåÆ AL format enum |

## Key Functions / Methods

### Constructor `AudioPlayer(uint32_t rate, bool stereo, AudioFormat audioFormat)`
- **Purpose:** Initialize player with audio parameters and call `Init()`
- **Inputs:** Sample rate, stereo flag, audio format (8/16-bit, 32-bit float)
- **Side effects:** Stores queued_rate, queued_format, queued_stereo; calls Init()

### `Init(uint32_t rate, bool stereo, AudioFormat audioFormat)`
- **Purpose:** Store audio parameters locally
- **Inputs:** Sample rate, stereo flag, audio format
- **Side effects:** Sets rate, stereo, format members

### `AssignSource()`
- **Purpose:** Request an available OpenAL source from the manager and initialize it
- **Outputs/Return:** true if source obtained and initialized
- **Side effects:** Calls `OpenALManager::Get()->PickAvailableSource()`, then `SetUpALSourceInit()`
- **Notes:** Early return if source already assigned

### `ResetSource()`
- **Purpose:** Stop playback and detach all buffers from the current source
- **Side effects:** Calls `alSourceStop()`, `alSourcei(..., AL_BUFFER, 0)`, marks all buffers unused
- **Notes:** Early return if no source assigned

### `RetrieveSource()`
- **Purpose:** Stop source and transfer ownership back to caller
- **Outputs/Return:** `std::unique_ptr<AudioSource>` (moved)
- **Side effects:** Calls `ResetSource()`, moves `audio_source` to caller

### `UnqueueBuffers()`
- **Purpose:** Unqueue finished buffers and mark them available
- **Side effects:** Calls `alSourceUnqueueBuffers()`, updates buffer usage flags
- **Notes:** Queries `AL_BUFFERS_PROCESSED` to determine count

### `FillBuffers()`
- **Purpose:** Stream audio data into queued buffers; handle format changes
- **Calls:** `UnqueueBuffers()`, `HasBufferFormatChanged()`, `GetNextData()` (virtual), `alBufferData()`, `alSourceQueueBuffers()`
- **Side effects:** Updates queued_rate/format/stereo if format changed; fills buffers with audio data
- **Notes:** Stalls buffer updates if format changed and buffers are still queued (OpenAL limitation)

### `HasBufferFormatChanged()` const
- **Purpose:** Compare queued format with desired format from subclass
- **Outputs/Return:** true if any parameter diverged
- **Calls:** `GetAudioFormat()` (virtual)

### `Play()`
- **Purpose:** Prepare buffers and start playback
- **Outputs/Return:** true if playback initiated or already playing, false if no buffers queued (finished)
- **Calls:** `FillBuffers()`, `IsPlaying()`, `alSourcePlay()`, `alGetError()`
- **Side effects:** Calls OpenAL playback functions
- **Notes:** Early exit if no buffers queued = end of stream

### `Rewind()`
- **Purpose:** Stop playback and clear queued audio
- **Calls:** `ResetSource()`
- **Side effects:** Clears rewind_signal flag

### `IsPlaying()` const
- **Purpose:** Query current playback state
- **Outputs/Return:** true if source in PLAYING or PAUSED state
- **Calls:** `alGetSourcei(..., AL_SOURCE_STATE, ...)`
- **Notes:** PAUSED counts as "playing" (underrun treated as active)

### `Update()`
- **Purpose:** Check for parameter updates and rewind requests; sync with OpenAL
- **Outputs/Return:** true if successful
- **Calls:** `LoadParametersUpdates()` (virtual), `Rewind()` if signaled, `SetUpALSourceIdle()`
- **Side effects:** Updates is_sync_with_al_parameters; clears rewind_signal
- **Notes:** Called once per frame from audio thread

### `SetUpALSourceIdle()`
- **Purpose:** Update dynamic AL parameters (volume) when player processes
- **Outputs/Return:** `SetupALResult` (success, full_sync)
- **Calls:** `OpenALManager::Get()->GetMasterVolume()`, `alSourcef()` for AL_MAX_GAIN and AL_GAIN
- **Side effects:** Updates OpenAL source gain

### `SetUpALSourceInit()`
- **Purpose:** Initialize all AL source parameters on assignment
- **Outputs/Return:** true if no OpenAL error
- **Calls:** `alSourcei()` and `alSource3i()` (15+ calls), OpenALManager extension checks
- **Side effects:** Sets pitch, position, distance model, rolloff, filters, spatialize extensions
- **Notes:** Handles optional extensions (spatialization, direct channel remix)

## Control Flow Notes
**Lifecycle phases:**
1. **Construction** ΓåÆ `Init()` stores audio format
2. **Assignment** ΓåÆ `AssignSource()` requests source from manager, calls `SetUpALSourceInit()`
3. **Update loop** ΓåÆ `Update()` processes parameter changes, fires `Rewind()` if signaled
4. **Playback** ΓåÆ `Play()` fills buffers and starts playback
5. **Streaming** ΓåÆ Each frame: `UnqueueBuffers()` ΓåÆ `FillBuffers()` (calls virtual `GetNextData()`) ΓåÆ `alSourceQueueBuffers()`
6. **Stop/Reset** ΓåÆ `Rewind()` or `RetrieveSource()` clean up

Subclasses implement `GetNextData()` (fill buffer with decoded samples), `GetAudioFormat()` (return current audio properties), and `LoadParametersUpdates()` (detect format/rate changes).

## External Dependencies
- **OpenAL:** `<AL/al.h>`, `<AL/alext.h>` ΓÇö OpenAL core API (source/buffer management, state queries)
- **OpenALManager** ΓÇö Singleton managing device, context, source pool, master volume, listener position
- **Decoder.h** ΓÇö Audio decoding interface (included in header, used by subclasses)
- **Boost:** `boost::lockfree::spsc_queue`, `boost::unordered_map` ΓÇö Thread-safe queues and hash maps
- **Standard library:** `<array>`, `<atomic>`, `<algorithm>`, `<unordered_map>`, `<memory>` (unique_ptr)
