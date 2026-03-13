# Source_Files/Sound/StreamPlayer.h

## File Purpose
Defines `StreamPlayer`, a subclass of `AudioPlayer` that implements streaming audio playback via user-supplied callback functions. Designed for exclusive use by `OpenALManager` to play dynamic audio sources (noted for intro video playback).

## Core Responsibilities
- Implement callback-driven audio data provisioning
- Override `GetNextData()` to fetch audio samples from user callback
- Store and manage callback function pointer and opaque user data
- Declare fixed priority level for audio scheduling
- Integrate with OpenAL audio system through `AudioPlayer` base class

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CallBackStreamPlayer` | typedef (function pointer) | Signature: `int (*)(uint8* data, uint32_t length, void* userdata)` for on-demand audio frame delivery |
| `StreamPlayer` | class | Streaming audio player backed by callback functions |

## Global / File-Static State
None.

## Key Functions / Methods

### StreamPlayer (constructor)
- Signature: `StreamPlayer(CallBackStreamPlayer callback, uint32_t rate, bool stereo, AudioFormat audioFormat, void* userdata)`
- Purpose: Initialize streaming audio player with callback-based data source
- Inputs: Callback function pointer, sample rate, stereo flag, audio format, opaque user data
- Outputs/Return: None (constructor)
- Side effects: Initializes `AudioPlayer` base; stores callback and userdata members
- Calls: `AudioPlayer` base constructor
- Notes: Comment warns against use outside `OpenALManager`; constructor public only for `std::make_shared` compatibility; see `friend` declaration

### GetPriority
- Signature: `float GetPriority() const override`
- Purpose: Return scheduling priority for audio processing queue
- Inputs: None
- Outputs/Return: `10.f`
- Side effects: None
- Calls: None
- Notes: Hard-coded constant; comment indicates priority immaterial for intro video use case

### GetNextData
- Signature: `uint32_t GetNextData(uint8* data, uint32_t length) override`
- Purpose: Fetch next audio frame chunk (pure virtual override)
- Inputs: Buffer pointer, requested byte length
- Outputs/Return: Actual bytes written to buffer
- Side effects: Invokes user callback; may trigger external I/O or state changes
- Calls: `CallBackFunction(data, length, userdata)`
- Notes: Implementation in corresponding `.cpp` file; delegates entirely to callback

## Control Flow Notes
Integrated into `AudioPlayer`'s frame-update loop (called by `OpenALManager`). Each audio update cycle, `AudioPlayer::FillBuffers()` ΓåÆ `GetNextData()` ΓåÆ user callback supplies raw sample data, which OpenAL streams to audio hardware.

## External Dependencies
- `AudioPlayer.h` ΓÇö Base class providing OpenAL source management, atomic buffer queuing, and format mapping
- OpenAL headers (`AL/al.h`, `AL/alext.h`) ΓÇö Via `AudioPlayer`
- `Decoder.h` ΓÇö Provides `AudioFormat` enum (via `AudioPlayer`)
- `boost/lockfree/spsc_queue.hpp`, `boost/unordered_map.hpp` ΓÇö Via `AudioPlayer` for lock-free structures and hash maps
