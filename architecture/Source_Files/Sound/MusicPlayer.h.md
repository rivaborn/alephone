# Source_Files/Sound/MusicPlayer.h

## File Purpose
MusicPlayer extends AudioPlayer to manage dynamic music playback with support for sequences of audio segments, transitions, and fade effects. It handles smooth switching between sequences with configurable crossfading and fade-in/fade-out timing, thread-safe parameter updates, and integration with OpenAL audio rendering.

## Core Responsibilities
- Manage multiple music sequences, each containing multiple audio segments (decoders)
- Process audio data with transition and fade effects (linear, sinusoidal)
- Handle asynchronous sequence transition requests with crossfading logic
- Maintain thread-safe parameter updates (volume, loop) via atomic structures
- Compute transition offsets and apply fade envelopes to audio buffers
- Override AudioPlayer hooks for OpenAL source setup and audio data generation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `MusicParameters` | struct | Holds volume and loop settings for playback |
| `FadeType` | enum class | Specifies fade curve: None, Linear, or Sinusoidal |
| `Segment` | nested class | Wraps a StreamDecoder with outbound edge transitions mapped by sequence index |
| `Segment::Transition` | struct | Fade type and duration (seconds) for a fade envelope |
| `Segment::Edge` | struct | Defines a segment transition: target segment ID, fade-out/fade-in params, crossfade flag |
| `Sequence` | nested class | Vector of Segments; provides index-based segment access |
| `SegmentTransitionOffsets` | typedef | Pair of optional uint32_t (fade-out and fade-in sample offsets) |

## Global / File-Static State
None.

## Key Functions / Methods

### Constructor
- Signature: `MusicPlayer(std::vector<Sequence>& sequences, uint32_t starting_sequence_index, uint32_t starting_segment_index, const MusicParameters& parameters)`
- Purpose: Initialize music player with sequences, starting playback position, and audio parameters
- Inputs: Sequence vector reference, starting sequence/segment indices, MusicParameters
- Outputs/Return: None
- Side effects: Stores sequences, initializes current decoder and playback state
- Notes: Comment states must not be used outside OpenALManager; public for `make_shared` compatibility

### RequestSequenceTransition
- Signature: `bool RequestSequenceTransition(uint32_t sequence_index)`
- Purpose: Asynchronously request a transition to a new sequence
- Inputs: Target sequence index
- Outputs/Return: bool (success/failure)
- Side effects: Updates `requested_sequence_index` atomic; processed on next `GetNextData()` call
- Calls: (defined in implementation)

### GetNextData
- Signature: `uint32_t GetNextData(uint8* data, uint32_t length) override`
- Purpose: Fill audio buffer with decoded/processed segment data, handling transitions and fades
- Inputs: Pointer to audio buffer, requested length in bytes
- Outputs/Return: Number of bytes written
- Side effects: May invoke `ProcessTransition()`, `ApplyFade()`, `CrossFadeMix()`, `SwitchSegment()`; updates internal playback state
- Calls: `ProcessTransition()`, `SwitchSegment()`, `GetTransitionSequenceIndex()`

### ApplyFade
- Signature: `void ApplyFade(FadeType fade_type, bool fade_in, uint32_t fade_length, uint32_t current_position, uint8* data, uint32_t faded_data_length)`
- Purpose: Apply linear or sinusoidal fade envelope to audio samples
- Inputs: Fade type, direction (in/out), fade length, current sample position, audio buffer, fade data length
- Outputs/Return: void; modifies audio buffer in-place
- Side effects: Alters sample amplitudes

### CrossFadeMix
- Signature: `void CrossFadeMix(uint8* data_out, uint8* data_in, uint32_t length)`
- Purpose: Blend two audio streams (fade out + fade in) for smooth segment transitions
- Inputs: Output buffer, input buffer, length in bytes
- Outputs/Return: void; combines data into `data_out`
- Side effects: Overwrites output buffer

### ProcessTransition / ProcessTransitionIn
- Signature: `bool ProcessTransition(uint8* data, uint32_t length, uint32_t next_sequence_index, std::optional<Segment::Edge> segment_edge)` and `bool ProcessTransitionIn(...)`
- Purpose: Orchestrate fade-out and fade-in logic during segment/sequence transitions
- Inputs: Audio buffer, length, target sequence index, optional edge definition
- Outputs/Return: bool (transition complete)
- Side effects: May trigger `CrossFadeMix()`, `SwitchSegment()`
- Calls: `ComputeTransitionOffsets()`, `ProcessTransitionIn()`

### SwitchSegment
- Signature: `void SwitchSegment(std::optional<Segment::Edge> segment_edge, std::optional<SegmentTransitionOffsets> transition_offsets)`
- Purpose: Update internal state to playback a new segment (update decoder, indices, edge info)
- Inputs: Optional edge transition, optional offset pair
- Outputs/Return: void
- Side effects: Updates `current_decoder`, `current_segment_index`, `transition_is_active`, `current_transition_edge_offsets`

### GetPriority
- Signature: `float GetPriority() const override`
- Purpose: Return audio priority for mixing (fixed 5.0f above max volume)
- Outputs/Return: 5.0f
- Notes: Ensures music is prioritized over sound effects in mixdown

### UpdateParameters / GetParameters
- Signature: `void UpdateParameters(const MusicParameters& musicParameters)` / `MusicParameters GetParameters() const`
- Purpose: Thread-safe get/set of volume and loop parameters
- Inputs/Outputs: MusicParameters struct
- Side effects: `UpdateParameters` queues change via `AtomicStructure::Store()`; consumed in `LoadParametersUpdates()`

## Control Flow Notes
**Initialization:** MusicPlayer is constructed with all sequences and a starting sequence/segment index; playback begins immediately upon OpenAL source assignment.

**Per-frame:** `GetNextData()` is called by AudioPlayer's update loop to fill audio buffers. Within `GetNextData()`, pending sequence transitions (from `RequestSequenceTransition()` calls) are processed. If a transition is pending, `ProcessTransition()` is invoked to fade out the current segment and crossfade into the target segment.

**Transition & Fading:** Transitions between segments use optional crossfading (both segments playing briefly together). Fade-in and fade-out durations are specified per edge. `ApplyFade()` modulates sample amplitude linearly or sinusoidally.

**Parameter updates:** `LoadParametersUpdates()` (called by base AudioPlayer) consumes queued parameter changes from the lock-free queue, ensuring thread-safe updates without blocking the audio thread.

## External Dependencies
- **AudioPlayer.h**: Base class; provides audio thread integration, OpenAL source management, buffer queueing
- **Decoder.h** (implied): `StreamDecoder` abstract class for decoding audio formats
- **OpenAL**: `<AL/al.h>`, `<AL/alext.h>` for audio output (via base class)
- **Boost**: `boost::lockfree::spsc_queue` for thread-safe parameter updates
- **Standard Library**: `<unordered_map>`, `<vector>`, `<atomic>`, `<optional>`, `<memory>`, `<utility>`
