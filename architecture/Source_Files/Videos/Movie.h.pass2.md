# Source_Files/Videos/Movie.h - Enhanced Analysis

## Architectural Role

Movie.h implements a **gameplay video recording system** that captures rendered frames and audio into an encoded video file via libav/ffmpeg. It sits at the convergence of three major subsystemsΓÇö**RenderMain** (framebuffer capture), **Sound** (audio synchronization), and **Files** (disk output)ΓÇöand decouples the potentially expensive encoding operation onto a background thread via producer-consumer queues. This allows 30 FPS gameplay to proceed without stalling on codec overhead.

## Key Cross-References

### Incoming (who depends on this file)
- **RenderMain/render.cpp** ΓåÆ calls `AddFrame()` once per rendered frame to queue video data
- **Sound subsystem** ΓåÆ queries `GetCurrentAudioTimeStamp()` to synchronize audio encoding with captured video timestamps
- **Shell/UI** ΓåÆ calls `PromptForRecording()` / `StartRecording()` / `StopRecording()` / `IsRecording()` for recording lifecycle control
- **Main loop (marathon2.cpp)** ΓåÆ likely calls `AddFrame()` each game tick

### Outgoing (what this file depends on)
- **RenderMain/OGL_FBO.h** ΓåÆ conditional `#ifdef HAVE_OPENGL` framebuffer object for GPU framebuffer readback (modern rendering path)
- **libav/ffmpeg** (opaque `struct libav_vars`)ΓåÆ video/audio encoding, muxing, file I/O
- **SDL2** ΓåÆ `SDL_Thread`, `SDL_sem`, `SDL_Surface`, `SDL_Rect` for threading, synchronization, software framebuffer fallback
- **cseries.h** ΓåÆ base types (`uint8`, `uint64_t`), platform abstraction

## Design Patterns & Rationale

| Pattern | Where | Why |
|---------|-------|-----|
| **Singleton (lazy, non-thread-safe)** | `instance()` | Single global recording state; avoids global variable; matches engine idiom |
| **Producer-Consumer** | `video_queue`, `audio_queue` | Decouple main thread frame capture from background encoding; maintain realtime 30 FPS |
| **RAII (unique_ptr)** | `StoredFrame`, `frameBufferObject` | Automatic memory cleanup; modern C++11 style |
| **Semaphore synchronization** | `encodeReady`, `fillReady` | Prevent queue overflow (main thread blocks if encoder falls behind) and starvation (encoder waits for frames) |
| **Enum for extensibility** | `FrameType` enum | Mark special frames (fade, chapter transitions) for codec keyframe hints or metadata |

The **queue + semaphore** design is idiomatic to realtime engines circa 2012 (pre-C++17 structured concurrency). It's safer than raw mutexes but more low-level than modern `std::condition_variable`. The producer-consumer split keeps encoding latency off the critical path.

## Data Flow Through This File

```
Game Loop (30 FPS)
Γö£ΓöÇ Render frame ΓåÆ [framebuffer memory]
Γö£ΓöÇ AddFrame() 
Γöé  Γö£ΓöÇ Read framebuffer (SDL_Surface or FBO)
Γöé  ΓööΓöÇ Enqueue StoredFrame(buffer, ts, duration, keyframe) ΓåÆ video_queue
Γöé  ΓööΓöÇ Signal encodeReady semaphore
ΓööΓöÇ Audio system updates current_audio_timestamp

Background Encoding Thread
Γö£ΓöÇ Wait on encodeReady semaphore (or polling)
Γö£ΓöÇ DequeueFrames()
Γöé  Γö£ΓöÇ Pop from video_queue, audio_queue in timestamp order
Γöé  Γö£ΓöÇ Call EncodeVideo()/EncodeAudio() ΓåÆ libav context
Γöé  ΓööΓöÇ Write compressed frames to output file
Γö£ΓöÇ Signal fillReady semaphore (notify main thread queue has space)
ΓööΓöÇ Exit on StopRecording() signal
```

**Key state transitions:**
- **Idle** ΓåÆ `StartRecording()` (setup buffers, spawn thread) ΓåÆ **Recording** ΓåÆ `StopRecording()` (flush queues, finalize codec, join thread) ΓåÆ **Idle**
- Timestamp synchronization: `last_written_timestamp` tracks output position; `current_audio_timestamp` keeps audio system in sync for realtime A/V alignment

## Learning Notes

**What developers learn from this file:**
- **Realtime constraint handling**: Even simple tasks (write compressed video) can't block the 30 FPS loop; queueing + background threads solve this
- **A/V synchronization is nontrivial**: Separate video/audio pipelines with different capture rates require explicit timestamp tracking and possibly frame reordering
- **Platform abstraction at subsystem boundaries**: Uses SDL2 for threading, conditionally includes OpenGL FBO, falls back to SDL_Surface software framebuffer
- **Enum-driven extensibility**: `FrameType` suggests future work (chapters, fade detection) without redesign

**Idiomatic to this era (2012ΓÇô2015):**
- Hybrid C++11 (unique_ptr, std::vector, std::string) with older SDL2 thread APIs (not std::thread)
- Opaque `struct libav_vars` hides codec complexity (typical for third-party integration)
- Semaphore-based synchronization predates C++17 structured concurrency and `std::stop_token`

## Potential Issues

1. **Race in `instance()`** (line 39ΓÇô41): Non-atomic lazy initialization without locking. If two threads call `instance()` simultaneously on first call, both could allocate separate Movie objects. Mitigation: call once from main thread on startup, or use Meyer's singleton pattern.

2. **Unsynchronized `current_audio_timestamp`** (line 125): Sound subsystem likely writes this; encoding thread reads it for sync. No visible lock ΓåÆ potential data race.

3. **Thread lifecycle safety**: `encodeThread`, `encodeReady`, `fillReady` are raw pointers/handles. If Movie is destroyed while encoding is active, crash likely. Missing destructor.

4. **FBO lifecycle tied to instance**: `frameBufferObject` created in Setup() but conditionally compiled; unclear if it's re-created per recording session or persists across multiple StartRecording() calls.

---

**Integration insight**: Movie.h is a **thin facade** around libav; the real encoding logic is in `.cpp` (DequeueFrames, EncodeVideo, EncodeAudio). This header exposes only the public gameplay-facing API, keeping codec details opaqueΓÇöappropriate for a modular engine.
