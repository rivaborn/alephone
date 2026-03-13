# Source_Files/Sound/MusicPlayer.cpp

## File Purpose
Implementation of MusicPlayer, an audio player specializing in structured music with segment-based playback, transitions, and crossfading. Manages complex audio state transitions between music sequences/segments with configurable fade effects.

## Core Responsibilities
- Decode and buffer audio data from structured music segments
- Compute and apply fade-in/fade-out transitions with linear or sinusoidal envelopes
- Mix crossfade audio when transitioning between segments with compatible formats
- Determine next playable segment and handle automatic transitions
- Process thread-safe transition requests from main thread during playback
- Configure OpenAL source properties (gain, volume) for music playback
- Track decoder positions across segment switches, especially for same-decoder crossfades

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MusicParameters` | struct | Volume and loop settings; wrapped in AtomicStructure for thread-safe updates |
| `FadeType` | enum | None, Linear, Sinusoidal fade types |
| `Segment` | nested class | Represents a single audio segment with decoder and edges to other segments |
| `Segment::Edge` | struct | Defines transition target, fade-out/in settings, and crossfade flag |
| `Segment::Transition` | struct | Fade type and duration (seconds) for a directional transition |
| `Sequence` | nested class | Collection of segments forming a playable music sequence |
| `SegmentTransitionOffsets` | typedef | Pair of optional byte offsets (fade-out position, fade-in position) |

## Global / File-Static State
None.

## Key Functions / Methods

### MusicPlayer (constructor)
- **Signature:** `MusicPlayer(std::vector<Sequence>& sequences, uint32_t starting_sequence_index, uint32_t starting_segment_index, const MusicParameters& parameters)`
- **Purpose:** Initialize player with starting segment and music parameters; delegates to parent AudioPlayer constructor with decoder properties
- **Inputs:** Sequence vector, starting indices, music parameters (volume, loop)
- **Outputs:** Initialized MusicPlayer instance
- **Side effects:** Stores sequences, current decoder, indices; calls decoder Rewind()
- **Calls:** AudioPlayer constructor, `GetSegment()`, `GetDecoder()`, `Rewind()`

### GetNextData
- **Signature:** `uint32_t GetNextData(uint8* data, uint32_t length) override`
- **Purpose:** Main audio decode loop; fetches next chunk, applies transitions, switches segments when needed
- **Inputs:** Output data buffer, requested length in bytes
- **Outputs:** Actual bytes decoded/mixed
- **Side effects:** Updates transition indices, processes transitions via ProcessTransition(), recursively fills buffer if format changes or segment boundary crossed
- **Calls:** `GetTransitionSequenceIndex()`, `Decode()`, `ProcessTransition()`, `SwitchSegment()`, `HasBufferFormatChanged()`
- **Notes:** Recursive if buffer format changed; handles end-of-segment transitions

### ProcessTransition
- **Signature:** `bool ProcessTransition(uint8* data, uint32_t length, uint32_t next_sequence_index, std::optional<Segment::Edge> segment_edge)`
- **Purpose:** Apply fade-out effect and optionally mix crossfade data when transitioning to next segment
- **Inputs:** Audio buffer, length, target sequence, optional edge descriptor
- **Outputs:** bool indicating if transition is still active
- **Side effects:** Modifies audio buffer in-place; may decode and position-shift next decoder during crossfade
- **Calls:** `ComputeTransitionOffsets()`, `ApplyFade()`, `ProcessTransitionIn()`, `CrossFadeMix()`, `Decode()`
- **Notes:** Handles same-decoder crossfades by tracking `crossfade_same_decoder_current_position`; complex position arithmetic

### ProcessTransitionIn
- **Signature:** `bool ProcessTransitionIn(uint8* data, uint32_t length, uint32_t next_sequence_index, std::pair<Segment::Edge, SegmentTransitionOffsets> segment_edge_offsets)`
- **Purpose:** Apply fade-in effect to incoming segment audio data during crossfade
- **Inputs:** Buffer (incoming fade-in audio), length, target sequence, transition edge and offsets
- **Outputs:** bool indicating if fade-in is still in progress
- **Side effects:** Modifies buffer via ApplyFade()
- **Calls:** `GetSegment()`, `GetDecoder()`, `ApplyFade()`

### ApplyFade
- **Signature:** `void ApplyFade(FadeType fade_type, bool fade_in, uint32_t fade_total_length, uint32_t current_position, uint8* data, uint32_t faded_data_length)`
- **Purpose:** Apply fade envelope (linear or sinusoidal) to float audio samples
- **Inputs:** Fade type, direction (in/out), total fade duration, current sample position, sample buffer, samples to fade
- **Outputs:** None (in-place modification)
- **Side effects:** Multiplies each sample by fade factor; asserts 4-byte float alignment
- **Calls:** `std::sin()`, `std::cos()` for sinusoidal; uses `M_PI_2` constant
- **Notes:** Linear: `t` or `1-t`; Sinusoidal: `sin(t*╧Ç/2)` or `cos(t*╧Ç/2)` where t Γêê [0,1]

### CrossFadeMix
- **Signature:** `void CrossFadeMix(uint8* data_out, uint8* data_in, uint32_t length)`
- **Purpose:** Mix fade-out and fade-in sample streams for smooth crossfading
- **Inputs:** Output (fade-out) buffer, input (fade-in) buffer, length in bytes
- **Outputs:** None (modifies data_out in-place)
- **Side effects:** Clamps summed samples to [-1.0f, 1.0f] to prevent distortion
- **Calls:** `std::clamp()`

### SwitchSegment
- **Signature:** `void SwitchSegment(std::optional<Segment::Edge> segment_edge, std::optional<SegmentTransitionOffsets> transition_offsets)`
- **Purpose:** Advance to next segment/sequence; update decoder and playback state
- **Inputs:** Optional edge (transition definition), optional offsets (for fade timing)
- **Outputs:** None
- **Side effects:** Updates `current_decoder`, sequence/segment indices, transition edge cache, resets crossfade position tracking; calls `Init()` to notify parent of format change
- **Calls:** `Position()`, `Init()`
- **Notes:** Restores crossfade position before switching to handle same-decoder reuse

### ComputeTransitionOffsets
- **Signature:** `SegmentTransitionOffsets ComputeTransitionOffsets(uint32_t sequence_index, const Segment::Edge& segment_edge) const`
- **Purpose:** Calculate byte offsets where fade-out and fade-in should occur during segment transition
- **Inputs:** Target sequence index, segment edge (defines transition times)
- **Outputs:** Pair of optional byte positions (fade-out start, fade-in end)
- **Side effects:** None
- **Calls:** `GetSegment()`, `GetDecoder()`, `GetAudioFormat()`, `Duration()`
- **Notes:** Validates decoder format compatibility; checks that durations exceed fade times before computing offsets

### GetTransitionSequenceIndex
- **Signature:** `uint32_t GetTransitionSequenceIndex() const`
- **Purpose:** Determine which sequence to transition to based on requested sequence and current playback position
- **Inputs:** None (reads instance state and atomic requested_sequence_index)
- **Outputs:** Target sequence index
- **Side effects:** Reads atomic `requested_sequence_index`
- **Calls:** `GetSegment()`, `GetSegmentEdge()`, `ComputeTransitionOffsets()`
- **Notes:** Returns requested sequence only if reachable (edge exists and fade-out hasn't started)

### RequestSequenceTransition
- **Signature:** `bool RequestSequenceTransition(uint32_t sequence_index)`
- **Purpose:** Public API for main thread to request music sequence change during playback
- **Inputs:** Target sequence index
- **Outputs:** bool (success if index valid)
- **Side effects:** Atomic store to `requested_sequence_index`
- **Calls:** None (atomic operation)

### SetUpALSourceIdle
- **Signature:** `SetupALResult SetUpALSourceIdle() override`
- **Purpose:** Configure OpenAL source gain based on master and music volume multiplier
- **Inputs:** None (reads from OpenALManager and parameters)
- **Outputs:** SetupALResult (success indicator)
- **Side effects:** Calls `alSourcef()` to set AL_MAX_GAIN and AL_GAIN
- **Calls:** `OpenALManager::Get()`, `alSourcef()`, `alGetError()`

## Control Flow Notes
Playback runs on the audio thread in OpenALManager's decode callback loop:
1. **GetNextData()** fetches next chunk from current decoder
2. **GetTransitionSequenceIndex()** checks if main thread requested a sequence change
3. **ProcessTransition()** applies fade-out and/or mixes crossfade if edge exists
4. If segment exhausted, **SwitchSegment()** advances to next segment or loops
5. If format changed, recursively fills remainder of buffer
6. Returns mixed audio to OpenAL

Main thread calls **RequestSequenceTransition()** asynchronously; audio thread picks it up during GetTransitionSequenceIndex() checks and applies fade-in/out at computed offsets.

## External Dependencies
- **AudioPlayer** ΓÇö parent class; provides audio source management, format tracking
- **StreamDecoder** ΓÇö abstract decoder interface (Position, Decode, Rate, Duration, etc.)
- **OpenALManager** ΓÇö singleton for querying master/music volume
- **OpenAL** ΓÇö `alSourcef()`, `alGetError()` for source configuration
- **Standard library** ΓÇö `<optional>`, `<cmath>`, `<algorithm>`, `<memory>`, `<vector>`, `<atomic>`, `<utility>`
