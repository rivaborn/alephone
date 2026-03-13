# Source_Files/Videos/Movie.cpp

## File Purpose
Implementation of the Movie class for Aleph One game engine's WebM video export feature. Encodes game footage and audio into WebM files using a separate encoding thread, supporting both OpenGL and SDL rendering paths with VP8 video and Vorbis audio codecs.

## Core Responsibilities
- Video and audio frame capture from game rendering
- VP8 video encoding via libvpx with configurable bitrate and quality
- Vorbis audio encoding via libvorbis with configurable quality
- WebM/Matroska container creation and file I/O
- Thread-safe producerΓÇôconsumer architecture: main thread captures frames, encode thread processes them
- Frame synchronization and cluster-based multiplexing of audio/video streams
- Quality scaling based on resolution (YouTube-like bitrate presets)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `libav_vars` | struct | State container for all encoding libraries (matroska, vorbis, vpx); tracks frame counters and stream properties |
| `MovieFileWrapper` | class | RAII wrapper around SDL_RWops file handle; implements libmatroska's IOCallback interface |
| `StoredFrame` (inner class) | struct | Holds encoded frame data with timestamp, duration, and keyframe flag |
| `FrameQueue` | typedef | `std::queue<std::unique_ptr<StoredFrame>>`; separate queues for video and audio |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cpu_count` (in `get_cpu_count()`) | static int | function-static | Cached processor count used to configure vpx encoder thread count |

## Key Functions / Methods

### Constructor: `Movie::Movie()`
- **Signature:** `Movie::Movie()`
- **Purpose:** Initialize instance with default state; allocate `libav_vars` struct.
- **Inputs:** None
- **Outputs/Return:** Instance ready for `PromptForRecording()` or `StartRecording()`.
- **Side effects:** Allocates heap memory for `av` pointer; zeroes it.
- **Calls:** None visible.
- **Notes:** Singleton pattern via `instance()` static method.

### `Movie::Setup()`
- **Signature:** `bool Setup()`
- **Purpose:** Initialize all encoding pipelines: SDL surface, VP8 codec, Vorbis codec, matroska headers, thread synchronization.
- **Inputs:** None (reads from `moviefile`, preferences, OpenALManager).
- **Outputs/Return:** `true` if setup succeeds; `false` on any error (calls `ThrowUserError`).
- **Side effects:** Creates SDL surface, thread semaphores, encode thread; writes partial WebM headers to file.
- **Calls:** `get_fps_target()`, `get_cpu_count()`, `ScaleQuality()`, `SDL_CreateRGBSurface()`, `vpx_codec_enc_init()`, `vorbis_*()` functions, `SDL_CreateSemaphore()`, `SDL_CreateThread()`.
- **Notes:** Failure at any point calls `ThrowUserError()` and returns false; no partial state left behind. Allocates large buffers for video (4├ùview_rect.w├ùh+10KB) and audio (2 channels ├ù in_bps ├ù fps).

### `Movie::AddFrame()`
- **Signature:** `void AddFrame(FrameType ftype)`
- **Purpose:** Capture current frame (pixel + audio) from game and queue for encoding.
- **Inputs:** `ftype` ΓÇô frame type (FRAME_NORMAL, FRAME_FADE, FRAME_CHAPTER); FRAME_FADE skips capture if keyboard active.
- **Outputs/Return:** None.
- **Side effects:** Calls `Setup()` on first frame; captures pixels via `libyuv::ARGBToI420()` or OpenGL `glReadPixels()`; captures audio via `OpenALManager::GetPlayBackAudio()`. Signals encode thread via semaphore.
- **Calls:** `IsRecording()`, `Setup()`, `MainScreenIsOpenGL()`, `libyuv::ARGBToI420()`, `glBlitFramebufferEXT()`, `glReadPixels()`, `OpenALManager::Get()->GetPlayBackAudio()`, semaphore post.
- **Notes:** Blocks on `fillReady` semaphore to synchronize with encode thread; if OpenGL, uses FBO for viewport-scaled capture and handles upside-down pixel buffer.

### `Movie::EncodeThread()` & `Movie::EncodeVideo(bool last)`
- **Signature:** `void EncodeThread()` / `void EncodeVideo(bool last)`
- **Purpose:** Main encode thread loop; feed frames to VP8 encoder and dequeue compressed packets.
- **Inputs:** `last` ΓÇô whether this is final flush (nullptr image passed to encoder).
- **Outputs/Return:** None.
- **Side effects:** Calls `vpx_codec_encode()`, dequeues packets via `vpx_codec_get_cx_data()`, pushes `StoredFrame` entries into `video_queue`. Detects and skips out-of-order packets.
- **Calls:** `vpx_codec_encode()`, `vpx_codec_get_cx_data()`.
- **Notes:** `EncodeThread()` loops on `encodeReady` semaphore until `stillEncoding` is false; coordinates with `AddFrame()` via semaphore handoff. Video PTS monotonicity enforced via `prev_video_pts` check.

### `Movie::EncodeAudio(bool last)`
- **Signature:** `void EncodeAudio(bool last)`
- **Purpose:** Feed audio samples to Vorbis encoder and dequeue compressed packets.
- **Inputs:** `last` ΓÇô final flush indicator.
- **Outputs/Return:** None.
- **Side effects:** Converts audio buffer (int16/float/int32/uint8) to Vorbis-compatible float stereo; calls `vorbis_analysis_*()` and `vorbis_bitrate_flushpacket()`; pushes packets into `audio_queue`. Tracks `current_audio_timestamp` based on granule position.
- **Calls:** `vorbis_analysis_buffer()`, `vorbis_analysis_wrote()`, `vorbis_analysis_blockout()`, `vorbis_analysis()`, `vorbis_bitrate_addblock()`, `vorbis_bitrate_flushpacket()`, `vorbis_granule_time()`.
- **Notes:** Audio format (ALC_SHORT_SOFT, ALC_FLOAT_SOFT, etc.) selected by OpenAL at setup time; packet ordering enforced via `prev_audio_packetno`.

### `Movie::DequeueFrames(bool last)` & `Movie::DequeueFrame()`
- **Signature:** `void DequeueFrames(bool last)` / `void DequeueFrame(FrameQueue &queue, uint64_t tracknum, bool start_cluster)`
- **Purpose:** Manage frame queue ordering and write frames to WebM file respecting Matroska cluster/keyframe rules.
- **Inputs:** `last` ΓÇô final flush; `queue` ΓÇô video or audio queue; `tracknum` ΓÇô VIDEO_TRACK_NUMBER (1) or AUDIO_TRACK_NUMBER (2); `start_cluster` ΓÇô whether to finalize current cluster before writing.
- **Outputs/Return:** None.
- **Side effects:** Pops frames from queues, creates/finalizes KaxCluster objects, renders frames via libmatroska, updates seek head and cue points. Maintains `last_written_timestamp` and `total_duration`.
- **Calls:** libmatroska functions (`KaxBlockBlob`, `KaxCluster`, rendering).
- **Notes:** **Timing rules enforced**: (1) frame timecodes monotonically increase; (2) audio precedes video at same timecode; (3) keyframes start clusters; (4) audio during keyframe stays in same cluster. Complex logic handles overlapping audio/video intervals.

### `Movie::StopRecording()`
- **Signature:** `void StopRecording()`
- **Purpose:** Finalize recording: flush encode thread, finalize WebM file, clean up all resources.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Sets `stillEncoding = false`, waits for encode thread exit, flushes remaining video/audio, calls `DequeueFrames(true)`, updates duration and seek head in WebM, overwrites void placeholder with seek head, closes file, deallocates all libvorbis/libvpx/matroska objects, restores audio device mode.
- **Calls:** `SDL_WaitThread()`, `EncodeVideo(true)`, `EncodeAudio(true)`, `DequeueFrames(true)`, matroska rendering and finalization functions, `vorbis_*_clear()`, `vpx_codec_destroy()`.
- **Notes:** All cleanup is defensive: checks null pointers before freeing. Matroska file structure finalized by updating duration field and seek head in reserved void space.

### Utility: `ScaleQuality(int quality, int zeroLevel, int fiftyLevel, int hundredLevel)`
- **Signature:** `int ScaleQuality(int quality, int zeroLevel, int fiftyLevel, int hundredLevel)`
- **Purpose:** Linear interpolation for quality settings (0ΓÇô100 range).
- **Inputs:** `quality` (0ΓÇô100); three reference points.
- **Outputs/Return:** Interpolated integer value.
- **Side effects:** None.
- **Notes:** Used for VP8 quantizer, Vorbis quality, and encoding deadline.

### Utility: `get_cpu_count()`
- **Signature:** `static int get_cpu_count(void)`
- **Purpose:** Detect CPU core count for encoder thread configuration.
- **Inputs:** None.
- **Outputs/Return:** Number of processors (cached after first call).
- **Side effects:** Uses platform-specific syscalls (sysconf, sysctl, GetSystemInfo).

## Control Flow Notes

**Initialization Path:**
1. `PromptForRecording()` ΓåÆ file dialog ΓåÆ `StartRecording(path)`
2. `StartRecording()` sets `moviefile`, stops audio, toggles device mode for loopback.
3. Next `AddFrame()` calls `Setup()`: allocates surfaces, codecs, threads, writes matroska headers.

**Encoding Loop (each frame):**
1. Main thread: `AddFrame()` captures pixels/audio, waits on `fillReady`, signals `encodeReady`.
2. Encode thread: wakes on `encodeReady`, calls `EncodeVideo()` and `EncodeAudio()`, calls `DequeueFrames(false)` to write completed frames, signals `fillReady`.
3. Synchronization via SDL semaphores ensures no frame buffer overwrite during encoding.

**Shutdown Path:**
1. `StopRecording()` sets `stillEncoding = false`, wakes encode thread.
2. Encode thread exits its loop.
3. Main thread flushes final frames via `EncodeVideo(true)`, `EncodeAudio(true)`, `DequeueFrames(true)`.
4. Matroska file finalized: seek head overwrites reserved void, duration set, file closed.
5. All EBML/Vorbis/VPX resources freed.

## External Dependencies
- **libvorbis/libogg** ΓÇô Vorbis audio codec.
- **libvpx** ΓÇô VP8 video encoder.
- **libmatroska/libebml** ΓÇô WebM/Matroska container format.
- **libyuv** ΓÇô Color space conversion (ARGB ΓåÆ I420 YUV).
- **SDL2** ΓÇô `SDL_RWops` (file I/O), `SDL_CreateThread`, semaphores, surfaces.
- **OpenGL** ΓÇô Frame buffer capture (when `HAVE_OPENGL` and `MainScreenIsOpenGL()` true).
- **OpenALManager** ΓÇô Audio playback capture via loopback device; controls audio device mode.
- **alephversion.h** ΓÇô Version string for Matroska metadata.
- **Logging.h** ΓÇô `logError()` for debug output.
- **interface.h** ΓÇô `get_fps_target()`, `get_keyboard_controller_status()`, `alert_user()`, file dialogs.
- **screen.h** ΓÇô Screen instance, viewport info, surface access, OpenGL detection.
