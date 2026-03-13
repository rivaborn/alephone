# Source_Files/Videos/Movie.h

## File Purpose
Manages video and audio recording of gameplay to a file. Provides a singleton interface for starting/stopping recording, queueing frames, and coordinating encoding on a background thread using libav/ffmpeg.

## Core Responsibilities
- Singleton management of movie recording state
- User interface prompting for recording parameters
- Frame queuing and timestamp management
- Asynchronous video/audio encoding via background thread
- Buffer management for video and audio data
- Synchronization via semaphores between main and encoding threads
- Frame buffer object management for OpenGL capture (when available)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FrameType | enum | Distinguishes frame types: normal, fade, chapter |
| StoredFrame | nested class | Encapsulates a single frame with buffer, timestamp, duration, and keyframe flag |
| FrameQueue | typedef | Queue of `unique_ptr<StoredFrame>` for async processing |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| m_instance | Movie* | static (in instance()) | Singleton pointer |

## Key Functions / Methods

### instance()
- Signature: `static Movie *instance()`
- Purpose: Obtain singleton instance
- Inputs: None
- Outputs/Return: Pointer to Movie singleton
- Side effects: Creates singleton on first call
- Calls: None visible
- Notes: Non-thread-safe lazy initialization

### StartRecording()
- Signature: `void StartRecording(std::string path)`
- Purpose: Initialize recording to specified file path
- Inputs: path (output file path)
- Outputs/Return: None
- Side effects: Initializes buffers, creates encoding thread, resets timestamp state
- Calls: Setup(), Movie_EncodeThread() (inferred)
- Notes: Spawns background encoding thread

### StopRecording()
- Signature: `void StopRecording()`
- Purpose: Finalize and close current recording
- Inputs: None
- Outputs/Return: None
- Side effects: Signals encoding thread, waits for completion, closes file
- Calls: EncodeVideo(), EncodeAudio() (inferred, with last=true)
- Notes: Blocks until encoding thread finishes

### AddFrame()
- Signature: `void AddFrame(FrameType ftype = FRAME_NORMAL)`
- Purpose: Capture and queue current rendered frame for encoding
- Inputs: ftype (frame type; defaults to FRAME_NORMAL)
- Outputs/Return: None
- Side effects: Reads framebuffer, enqueues to video_queue, signals encoding thread
- Calls: Framebuffer read/copy (inferred)
- Notes: Called each render frame; uses semaphores to coordinate with encoding thread

### GetCurrentAudioTimeStamp()
- Signature: `uint64_t GetCurrentAudioTimeStamp()`
- Purpose: Retrieve current audio timestamp for A/V synchronization
- Inputs: None
- Outputs/Return: current_audio_timestamp
- Side effects: None
- Calls: None
- Notes: Queried by audio system to sync with video

**Notes:** `PromptForRecording()` and `IsRecording()` are straightforward UI/state query methods.

## Control Flow Notes
- **Recording start**: `StartRecording()` initializes buffers, spawns encoding thread via `Movie_EncodeThread()`
- **Frame capture**: During game loop, `AddFrame()` enqueues video frames; audio enqueued separately (mechanism not visible)
- **Async encoding**: Encoding thread dequeues from `video_queue` and `audio_queue`, writes to libav context
- **Shutdown**: `StopRecording()` signals end-of-stream, waits for thread completion
- **Synchronization**: Semaphores (`encodeReady`, `fillReady`) prevent queue overflow/starvation

## External Dependencies
- **SDL2**: `SDL_Thread`, `SDL_sem`, `SDL_Surface`, `SDL_Rect` for threading and surfaces
- **libav/ffmpeg**: `struct libav_vars` (opaque; implementation elsewhere)
- **OGL_FBO.h**: Frame buffer object for OpenGL capture (conditional `HAVE_OPENGL`)
- **cseries.h**: Base types, utilities, includes
- **Standard library**: `<memory>`, `<string>`, `<vector>`, `<queue>` for containers and RAII
