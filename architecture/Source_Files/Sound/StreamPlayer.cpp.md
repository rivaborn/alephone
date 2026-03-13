# Source_Files/Sound/StreamPlayer.cpp

## File Purpose
Implements `StreamPlayer`, a callback-driven audio player component for the Aleph One game engine. Audio data is fetched on-demand from an external callback function rather than stored in memory, enabling streaming of audio sources like intro videos.

## Core Responsibilities
- Construct StreamPlayer with callback function and playback parameters
- Override `GetNextData()` to pull audio frames from a registered callback
- Manage callback function pointer and user-defined context data
- Enforce callback invocation pattern with buffer size constraints

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CallBackStreamPlayer` | typedef (function pointer) | Callback signature: `int (*)(uint8* data, uint32_t length, void* userdata)` for audio data fetching |
| `StreamPlayer` | class | Audio player subclass that sources data via external callback |

## Global / File-Static State
None.

## Key Functions / Methods

### StreamPlayer (constructor)
- **Signature:** `StreamPlayer(CallBackStreamPlayer callback, uint32_t rate, bool stereo, AudioFormat audioFormat, void* userdata)`
- **Purpose:** Initialize a callback-driven audio player
- **Inputs:** callback function pointer, sample rate, stereo flag, audio format, opaque userdata pointer
- **Outputs/Return:** (constructor)
- **Side effects:** Stores callback and userdata; invokes parent `AudioPlayer` constructor
- **Calls:** `AudioPlayer::AudioPlayer(rate, stereo, audioFormat)`
- **Notes:** Header comment: "Must not be used outside OpenALManager (public for make_shared)" ΓÇö restricted usage pattern for shared_ptr factory

### GetNextData
- **Signature:** `uint32_t GetNextData(uint8* data, uint32_t length)` (override)
- **Purpose:** Fetch next audio chunk via registered callback
- **Inputs:** `data` (buffer), `length` (requested byte count)
- **Outputs/Return:** Number of bytes written to buffer
- **Side effects:** None (delegates to callback)
- **Calls:** `CallBackFunction(data, std::min(length, buffer_samples), userdata)`
- **Notes:** Caps request size at `buffer_samples` (inherited from `AudioPlayer`); callback ownership/lifetime managed externally

## Control Flow Notes
No explicit init/frame/shutdown. Data is pulled on-demand by the audio system (likely OpenALManager) during playback updates. `GetNextData()` is called repeatedly as the audio buffer needs refilling.

## External Dependencies
- `AudioPlayer` (parent class via StreamPlayer.h)
- `std::min()` (standard library)
