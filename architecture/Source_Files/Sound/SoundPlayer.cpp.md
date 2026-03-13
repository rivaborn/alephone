# Source_Files/Sound/SoundPlayer.cpp

## File Purpose
Implements SoundPlayer, which manages individual sound playback via OpenAL. Handles 2D/3D spatial audio, volume/behavior transitions, rewinding, and audio format conversion (including mono-to-stereo for HRTF support).

## Core Responsibilities
- Initialize and manage sound playback parameters (pitch, volume, 3D position, obstruction flags)
- Simulate and compute sound volume based on distance and obstruction state
- Configure OpenAL audio sources for 2D panning and 3D spatial positioning
- Handle smooth parameter transitions when sound properties change during playback
- Process audio data with optional mono-to-stereo conversion for HRTF compatibility
- Manage sound rewinding (both soft and fast rewind modes)
- Apply behavior parameter tables based on obstruction/muffling conditions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SoundParameters | struct | Sound playback parameters: identifier, pitch, 2D/3D flag, obstruction flags, behavior type, source location |
| SoundBehavior | struct | Distance-based attenuation parameters: distance_reference, distance_max, rolloff_factor, max_gain, high_frequency_gain |
| SoundTransition | struct (nested) | Tracks smooth transitions: current_volume, current_sound_behavior, start_tick, allow_transition flag |
| Sound | struct | Audio data container: header (SoundInfo) and shared_ptr to audio data |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| sound_behavior_parameters[] | constexpr SoundBehavior[3] | static class member | Attenuation presets for unobstructed sounds (indexed by sound_behavior enum) |
| sound_obstructed_or_muffled_behavior_parameters[] | constexpr SoundBehavior[3] | static class member | Attenuation presets for single obstruction/muffling condition |
| sound_obstructed_and_muffled_behavior_parameters[] | constexpr SoundBehavior[3] | static class member | Attenuation presets for both obstruction and muffling |
| rewind_time | static constexpr uint32_t | class constant | Base rewind time in ticks (83U) |
| fast_rewind_time | static constexpr uint32_t | class constant | Fast rewind time in ticks (35U) |
| smooth_volume_transition_threshold | static constexpr float | class constant | Minimum volume delta to trigger smooth transition (0.1f) |
| smooth_volume_transition_time_ms | static constexpr uint32_t | class constant | Duration of smooth transitions in ms (300U) |

## Key Functions / Methods

### Simulate
- **Signature:** `float Simulate(const SoundParameters& soundParameters)` (static)
- **Purpose:** Predict the volume a sound would play at given parameters, using distance attenuation and obstruction state. Returns 0 if sound is out of range.
- **Inputs:** SoundParameters with 3D location, obstruction flags, behavior type, and listener reference
- **Outputs/Return:** Float volume [0.0, 1.0]; 0 means don't play
- **Side effects:** Calls OpenALManager::Get()->GetListener() to fetch current listener position
- **Calls:** OpenALManager::Get(), std::sqrt, std::pow, std::max
- **Notes:** Implements AL_INVERSE_DISTANCE_CLAMPED attenuation model. Selects behavior parameters based on obstruction/muffling flags.

### Init
- **Signature:** `void Init(const SoundParameters& parameters)`
- **Purpose:** Reinitialize the sound player with new parameters and audio header data.
- **Inputs:** SoundParameters
- **Outputs/Return:** None
- **Side effects:** Resets current_index_data, data_length, start_tick; calls AudioPlayer::Init()
- **Calls:** AudioPlayer::Init, SoundManager::GetCurrentAudioTick
- **Notes:** Called during construction and rewind.

### CanRewind
- **Signature:** `bool CanRewind(uint64_t baseTick) const`
- **Purpose:** Determine if a rewind request can be honored based on elapsed time and rewind mode.
- **Inputs:** baseTick (sound start time)
- **Outputs/Return:** Boolean; true if enough time has passed or soft-rewind is enabled
- **Side effects:** None
- **Calls:** OpenALManager::Get()->IsBalanceRewindSound(), CanFastRewind(), SoundManager::GetCurrentAudioTick()
- **Notes:** Soft rewind allows immediate rewind; hard rewind enforces minimum time delay.

### SetUpALSourceIdle
- **Signature:** `SetupALResult SetUpALSourceIdle()` (override)
- **Purpose:** Configure OpenAL source properties for current playback frame (pitch, gain, position, panning, distance model).
- **Inputs:** None (reads member parameters and audio_source)
- **Outputs/Return:** SetupALResult struct with success flag and transition-completion flag
- **Side effects:** Calls alSourcef/alSource3f/alSource3i multiple times; modifies OpenAL state
- **Calls:** ComputeVolumeForTransition (both overloads), OpenALManager::Get()->GetMasterVolume(), alSourcef, alSource3f, alSource3i
- **Notes:** Branches on is_2d flag. For 3D sounds, delegates to SetUpALSource3D(). Handles soft-stop and soft-start transitions.

### SetUpALSource3D
- **Signature:** `SetupALResult SetUpALSource3D()`
- **Purpose:** Configure OpenAL distance-attenuation, gain, and low-pass filter for 3D spatial audio.
- **Inputs:** None (reads parameters, obstruction flags, behavior tables)
- **Outputs/Return:** SetupALResult with success flag and behavior-transition flag
- **Side effects:** Calls alSourcef, alSourcei; modifies OpenAL low-pass filter state
- **Calls:** ComputeVolumeForTransition(SoundBehavior), OpenALManager::Get()->GetMasterVolume(), OpenALManager::Get()->GetLowPassFilter()
- **Notes:** Applies AL_INVERSE_DISTANCE_CLAMPED. Selects behavior parameters based on obstruction/muffling combo.

### ProcessData
- **Signature:** `uint32_t ProcessData(uint8_t* outputData, uint32_t remainingSoundDataLength, uint32_t remainingBufferLength)`
- **Purpose:** Copy audio samples from sound data to output buffer, optionally converting mono to stereo for HRTF.
- **Inputs:** Output buffer, remaining sound data bytes, remaining output buffer bytes
- **Outputs/Return:** Bytes written to output buffer
- **Side effects:** Increments current_index_data; may call ConvertMonoToStereo template
- **Calls:** MustDisableHrtf(), std::copy, ConvertMonoToStereo<uint8_t/int16_t/float>
- **Notes:** Uses template instantiation for format-specific mono-to-stereo conversion if HRTF disabling is required.

### ConvertMonoToStereo
- **Signature:** `template<typename T> static uint32_t ConvertMonoToStereo(const uint8_t* inputBytes, uint8_t* outputBytes, uint32_t remainingInputBytes, uint32_t remainingOutputBytes)`
- **Purpose:** Template function to duplicate mono samples into stereo pairs for HRTF processing.
- **Inputs:** Input/output byte buffers, byte counts
- **Outputs/Return:** Bytes written to output
- **Side effects:** Writes to outputBytes
- **Calls:** std::memcpy
- **Notes:** Static_assert ensures T is trivially copyable. Handles 8-bit, 16-bit, and 32-bit float formats.

### ComputeVolumeForTransition (float overload)
- **Signature:** `float ComputeVolumeForTransition(float targetVolume)`
- **Purpose:** Smoothly interpolate volume over a fixed transition time when volume changes significantly.
- **Inputs:** Target volume
- **Outputs/Return:** Interpolated volume for current frame
- **Side effects:** Tracks transition start time in sound_transition struct
- **Calls:** ComputeParameterForTransition(), SoundManager::GetCurrentAudioTick()
- **Notes:** Only triggers if abs(target - current) exceeds threshold (0.1f); transitions over 300ms.

### ComputeVolumeForTransition (SoundBehavior overload)
- **Signature:** `SoundBehavior ComputeVolumeForTransition(const SoundBehavior& targetSoundBehavior)`
- **Purpose:** Smoothly interpolate all five sound behavior parameters during transitions.
- **Inputs:** Target SoundBehavior struct
- **Outputs/Return:** Interpolated SoundBehavior for current frame
- **Side effects:** Updates sound_transition state; resets start_transition_tick when transition completes
- **Calls:** ComputeParameterForTransition() x5, SoundManager::GetCurrentAudioTick()
- **Notes:** Interpolates distance_reference, distance_max, rolloff_factor, max_gain, high_frequency_gain independently.

### LoadParametersUpdates
- **Signature:** `bool LoadParametersUpdates()` (override)
- **Purpose:** Consume all pending parameter updates from queues, select highest-priority viable update, and apply it.
- **Inputs:** None (drains atomic parameter queues)
- **Outputs/Return:** Boolean; true if any updates or soft-stop were applied
- **Side effects:** Drains parameters queue and rewind_parameters queue; updates member state
- **Calls:** Simulate() for each candidate parameter, parameters.Consume(), parameters.Set(), rewind_parameters.Consume(), rewind_parameters.Set()
- **Notes:** Selects update with highest volume priority; ignores lower-priority updates. Soft-stop always triggers return true.

### Rewind
- **Signature:** `void Rewind()` (override)
- **Purpose:** Handle sound rewindingΓÇöeither hard rewind (seek to start via AudioPlayer) or soft rewind (wait for completion).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls AudioPlayer::Rewind(), Init(), SetUpALSourceInit(); resets transition state if source identifier changed
- **Calls:** CanRewind(), IsPlaying(), AudioPlayer::Rewind(), sound.Update(), Init(), ResetTransition(), SetUpALSourceInit()
- **Notes:** Soft rewind only allows rewind after data is exhausted; hard rewind seeks immediately if enough time elapsed.

## Control Flow Notes
**Initialization:** Constructor ΓåÆ Init() chains to AudioPlayer::Init, storing audio header and parameters.

**Per-frame Update:** Called during audio thread processing:
1. LoadParametersUpdates() consumes pending parameter changes
2. SetUpALSourceIdle() configures OpenAL source with current parameters, applying smooth transitions
3. GetNextData() is called by AudioPlayer base class to fill audio buffers
4. ProcessData() handles audio data copying and optional mono-to-stereo conversion

**Rewinding:** AskRewind() queues rewind; Rewind() executes it on next update, subject to CanRewind() constraints.

**Transition:** ComputeVolumeForTransition() methods interpolate volume and behavior over 300ms when sound properties change.

## External Dependencies
- **Includes:** AudioPlayer.h, OpenALManager.h, SoundManager.h
- **OpenAL API:** alSourcef, alSourcei, alSource3f, alSource3i, alGetError (AL_* constants)
- **Defined elsewhere:** AudioPlayer (base class), OpenALManager::Get(), SoundManager::GetCurrentAudioTick(), AtomicStructure<T>, SetupALResult, SoundInfo, SoundData, LoadedResource
- **STL:** std::sqrt, std::pow, std::max, std::min, std::abs, std::copy, std::memcpy, std::tuple, std::get, std::tie
- **Math:** M_PI (from cmath via OpenALManager.h)
