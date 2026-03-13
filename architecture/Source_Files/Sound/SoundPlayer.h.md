# Source_Files/Sound/SoundPlayer.h

## File Purpose
Defines the `SoundPlayer` class that manages individual sound playback in the Aleph One game engine. Handles 3D audio positioning, volume transitions, sound parameter updates, and rewind functionality via OpenAL backend.

## Core Responsibilities
- Manage playback lifecycle for individual sound instances (initialization, parameter updates, stopping)
- Compute sound priority based on distance and volume for playback prioritization
- Handle smooth volume and parameter transitions over time
- Support 3D positioning with distance-based attenuation and obstruction effects
- Implement soft-start (fade-in) and soft-stop (fade-out) behaviors
- Support sound rewind with fast/soft rewind variants
- Convert mono audio to stereo when needed
- Update OpenAL source parameters based on computed audio properties

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `SoundStereo` | struct | Stereo panning parameters (global gain, per-channel gains, panning flag) |
| `SoundParameters` | struct | Complete configuration for a sound instance (identifiers, pitch, 2D/3D mode, position, stereo params, behavior) |
| `SoundBehavior` | struct | Audio attenuation behavior (distance reference/max, rolloff factor, max gain, HF absorption) |
| `Sound` | struct | Sound container holding header metadata and audio data pointer |
| `SoundTransition` | struct (private) | Internal state tracking smooth transitions (tick timing, current volume, transition permission) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rewind_time` | uint32_t | static constexpr | Duration (83 ms) for normal rewind operations |
| `fast_rewind_time` | uint32_t | static constexpr | Duration (35 ms) for fast rewind operations |
| `smooth_volume_transition_threshold` | float | static constexpr | Volume delta threshold (0.1) to trigger smooth transitions |
| `smooth_volume_transition_time_ms` | uint32_t | static constexpr | Duration (300 ms) for smooth volume transitions |
| `sound_behavior_parameters` | SoundBehavior[] | static constexpr | 3 behavior presets for normal unobstructed sounds (quiet, normal, loud) |
| `sound_obstructed_or_muffled_behavior_parameters` | SoundBehavior[] | static constexpr | 3 behavior presets for partially obstructed sounds |
| `sound_obstructed_and_muffled_behavior_parameters` | SoundBehavior[] | static constexpr | 3 behavior presets for fully obstructed sounds |

## Key Functions / Methods

### SoundPlayer (constructor)
- **Signature:** `SoundPlayer(const Sound& sound, const SoundParameters& parameters)`
- **Purpose:** Initializes a sound player for a specific sound with given parameters; note requires OpenALManager context
- **Inputs:** Sound object (header + data), SoundParameters configuration
- **Outputs/Return:** Constructed object
- **Side effects:** Initializes atomic structures, OpenAL source binding deferred to OpenALManager
- **Notes:** Constructor marked public for `make_shared` but intended for OpenALManager use only

### UpdateParameters
- **Signature:** `void UpdateParameters(const SoundParameters& parameters)`
- **Purpose:** Queue parameter updates for playback (non-blocking, thread-safe)
- **Inputs:** New SoundParameters
- **Outputs/Return:** void
- **Side effects:** Stores parameters in atomic queue for eventual consumption

### GetPriority
- **Signature:** `float GetPriority() const override`
- **Purpose:** Compute and return playback priority (higher = more important to play)
- **Inputs:** Current parameters (via `Simulate`)
- **Outputs/Return:** Priority float value
- **Calls:** `Simulate(parameters.Get())`
- **Notes:** Used by audio manager to prioritize playback when source count is limited

### Simulate (static)
- **Signature:** `static float Simulate(const SoundParameters& soundParameters)`
- **Purpose:** Compute priority metric for given parameters (likely distance-based volume)
- **Inputs:** SoundParameters configuration
- **Outputs/Return:** Priority float
- **Notes:** No side effects; deterministic based on parameters alone

### CanRewind
- **Signature:** `bool CanRewind(uint64_t baseTick) const`
- **Purpose:** Check if sound can perform normal rewind based on elapsed time
- **Inputs:** Base tick timestamp for timing calculation
- **Outputs/Return:** true if rewind is allowed
- **Notes:** Enforces `rewind_time` minimum between rewinds

### CanFastRewind
- **Signature:** `bool CanFastRewind(const SoundParameters& soundParameters) const`
- **Purpose:** Check if sound can perform fast rewind given current parameters
- **Inputs:** Current SoundParameters
- **Outputs/Return:** true if fast rewind allowed
- **Notes:** Likely checks `soft_rewind` flag and other parameter constraints

### HasActiveRewind
- **Signature:** `bool HasActiveRewind() const`
- **Purpose:** Query if rewind operation is currently in progress
- **Outputs/Return:** true if rewind active and not soft-stopping
- **Notes:** Checks both `rewind_signal` and `!soft_stop_signal`

### AskSoftStop
- **Signature:** `void AskSoftStop()`
- **Purpose:** Request graceful fade-out of sound (not supported for 3D sounds)
- **Side effects:** Sets `soft_stop_signal` atomic flag
- **Notes:** Triggers soft-stop via fade transition; 3D sounds have no fade behavior

### AskRewind
- **Signature:** `void AskRewind(const SoundParameters& soundParameters, const Sound& sound)`
- **Purpose:** Queue a rewind request with updated parameters and sound data
- **Inputs:** SoundParameters for current state, Sound object for new data
- **Side effects:** Updates rewind parameters/sound atomically
- **Notes:** Thread-safe queue-based API for main thread to audio thread communication

---

## Control Flow Notes
Fits into the audio playback pipeline as follows:
- **Initialization** (`Init`, `SetUpALSourceIdle`, `SetUpALSourceInit`): Called when OpenALManager assigns a source
- **Frame update** (`LoadParametersUpdates`, `GetNextData`, `ProcessData`): Called per audio buffer fill (~256 samples)
- **Parameter transitions**: Smooth transitions computed incrementally across multiple frames via `ComputeParameterForTransition`
- **Shutdown** (`Rewind`): Handles rewinding; destruction via OpenALManager

The class inherits from `AudioPlayer` and overrides virtual methods to provide sound-specific behavior (format queries, data fetching, parameter loading).

## External Dependencies
- **Includes:** `AudioPlayer.h` (base class), `SoundFile.h` (SoundInfo/SoundData types), `sound_definitions.h` (sound_behavior enum, world_location3d)
- **External symbols used:**
  - `AudioPlayer` (base class with OpenAL integration, atomic buffers)
  - `SoundInfo`, `SoundData` (sound metadata and PCM data)
  - `sound_behavior` enum (behavior categorization)
  - `world_location3d` (3D position type from engine world system)
  - `AtomicStructure<T>` (lock-free queue for thread-safe parameter passing)
  - OpenAL types: `ALuint` (implicitly via AudioPlayer)
