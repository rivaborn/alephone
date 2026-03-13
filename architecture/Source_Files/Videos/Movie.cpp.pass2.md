# Source_Files/Videos/Movie.cpp - Enhanced Analysis

## Architectural Role

This module is the **video export gateway** bridging the real-time game loop (RenderMain + Sound) to long-form WebM file output. It decouples frame capture from encoding via thread-safe semaphore-based handoff, allowing asynchronous VP8/Vorbis codec operation without stalling the 30 FPS game tick. Critically, it negotiates with **RenderMain** (pixel format, FBO handling, OpenGL context), **OpenALManager** (audio loopback device mode toggling), and **Preferences** (bitrate/quality settings) ΓÇö serving as a **coordination hub** for recording orchestration across previously-independent subsystems.

## Key Cross-References

### Incoming (who depends on this file)
- **RenderMain** (`screen.h`, `render.cpp`): Calls `AddFrame()` post-render to capture pixels via `glReadPixels()` or SDL surface read; queries `IsRecording()` to avoid redundant work during non-recording frames
- **Interface/Shell** (`interface.h`): Invokes `PromptForRecording()` to show file dialog; calls `StopRecording()` on shutdown; may call `IsRecording()` to grey out menu options
- **GameWorld** (implicit via interface): Recording lifetime spans a gameplay session; world state must remain deterministic during recording (no frame drops)
- **Preferences** (`graphics_preferences` global): Movie reads `movie_export_video_bitrate`, `movie_export_video_quality`, `movie_export_audio_quality` at setup time

### Outgoing (what this file depends on)
- **OpenALManager** (`OpenALManager.h`): Queries rendering format (ALC_SHORT_SOFT/FLOAT_SOFT/INT_SOFT/UNSIGNED_BYTE_SOFT), frequency (sample rate); toggles device mode to enable loopback capture; calls `GetPlayBackAudio()` each frame to drain captured samples
- **Files/Screen** (`screen.h`, `interface.h`): Calls `get_fps_target()` to determine timecode scale; uses `MainScreenIsOpenGL()` to branch pixel capture path; `FileSpecifier` for file dialog
- **CSeries** (`cseries.h`, `csalerts.h`, `Logging.h`): `alert_user()` for error dialogs; `logError()` for debug logging; `ThrowUserError()` is wrapper that calls both
- **LibMatroska/LibEBML** (`matroska/*.h`, `ebml/*.h`): Full Matroska container serialization; `KaxSegment`, `KaxCluster`, `KaxTracks`, `KaxCues` for WebM structure
- **LibVPX** (`vpx/vpx_encoder.h`): VP8 codec context and frame encoding
- **LibVorbis** (`vorbis/vorbisenc.h`): Vorbis audio codec and bitstream assembly

## Design Patterns & Rationale

**1. Producer-Consumer via Semaphores (Strict Handoff)**
- `fillReady` and `encodeReady` semaphores enforce exactly one frame in-flight at once
- Main thread blocks waiting for `fillReady` before capturing next frame; encode thread blocks waiting for `encodeReady` before processing
- Rationale: Avoids frame buffer ring; guarantees memory-efficient 1-frame-deep queue; encoder latency doesn't accumulate (tight synchronous coupling)
- Tradeoff: Can't pipeline multiple encodes; if encoder stalls, frame capture stalls (but acceptable for 30 FPS at typical encode speeds)

**2. Quality Interpolation (ScaleQuality)**
- 0ΓÇô100 slider maps to three anchor points (0%, 50%, 100%), then linearly interpolates between them
- Decouples user-facing "quality" from codec-specific parameters (VPX quantizer, Vorbis quality index, encoding deadline)
- Rationale: Hides codec-level tuning from UI; provides consistent quality curve across very different codec APIs
- Used for: VP8 CQ level, min/max quantizer, Vorbis VBR, encoding deadline

**3. Deferred Matroska Finalization**
- 4 KB void placeholder reserved upfront; seek head written **after** recording ends
- Duration and cue points calculated only after all frames encoded (timestamp monotonicity enforced in DequeueFrames)
- Rationale: Matroska seek head must appear early in file for quick seeking, but content (cues) appears late; rewriting void with seek head finalizes file structure
- Tradeoff: File must be seekable (SDL_RWops supports seek); can't stream to pipe

**4. Codec Negotiation at Setup Time**
- Movie queries OpenAL format (ALC_SHORT_SOFT ΓåÆ 16 bps, ALC_FLOAT_SOFT ΓåÆ 32 bps, etc.) once and caches it
- Rationale: Avoids per-frame format detection; assumes stable audio format across session (reasonable for game audio)
- Couples Movie tightly to OpenAL: if audio format changes mid-session, data corruption

**5. YouTube Bitrate Preset Hardcoding**
- Resolution-dependent bitrate map (4KΓåÆ40 Mbps, 1080pΓåÆ8 Mbps, etc.) plus fps scaling factor
- Rationale: Provides sensible defaults for web export without user tuning; baked-in knowledge of YouTube SDR recommendations
- Tradeoff: Users exporting to non-YouTube targets must manually override via preferences

## Data Flow Through This File

```
User triggers recording:
  PromptForRecording() ΓåÆ [file dialog] ΓåÆ StartRecording(path)
                        Γåô
                  moviefile = path
                  OpenALManager toggled to loopback mode
                        Γåô
  Next AddFrame():
    IsRecording() check
        Γåô
    Setup() [first call only]:
      - Allocate VP8 codec + Vorbis codec
      - Write Matroska headers (EBML head, segment, tracks)
      - Create encode thread + semaphores
        Γåô
    Capture pixels (OpenGL FBO or SDL surface)
    Γåô (libyuv::ARGBToI420)
    YUV420 ΓåÆ videobuf
        Γåô
    OpenALManager::GetPlayBackAudio() ΓåÆ audiobuf
        Γåô
    Signal encodeReady semaphore ΓåÆ [encode thread wakes]
        Γåô (main thread waits on fillReady)
    
  Encode thread loop:
    Wait on encodeReady semaphore
        Γåô
    EncodeVideo(): videobuf ΓåÆ VP8 codec ΓåÆ [encoded packets] ΓåÆ video_queue
    EncodeAudio(): audiobuf ΓåÆ Vorbis codec ΓåÆ [encoded packets] ΓåÆ audio_queue
        Γåô
    DequeueFrames(): pop video + audio frames, enforce timecode ordering, write to Matroska cluster
        Γåô
    Signal fillReady semaphore ΓåÆ [main thread wakes, captures next frame]
    
  User stops recording:
    StopRecording():
      - stillEncoding = false ΓåÆ [encode thread exits loop]
      - Flush VP8 (encode nullptr) + Vorbis ΓåÆ drain final packets
      - DequeueFrames(true) ΓåÆ finalize clusters
      - Rewrite seek head over void placeholder
      - Update duration field
      - Free all codec/Matroska/Vorbis structures
      - Close file
```

## Learning Notes

**Era-Specific Design Choices:**
- SDL semaphores instead of modern C++20 `std::latch` or `std::barrier`; suggests pre-C++20 target
- Manual `memset(av, 0, ...)` instead of constructor initialization; `libav_vars` is a legacy data struct, not a proper C++ class
- Direct `vpx_codec_t` and `vorbis_dsp_state` as struct members; no RAII wrappers; cleanup is manual (vorbis_*_clear, vpx_codec_destroy)

**Interesting Constraints:**
- Frame timecode scale fixed to `WEBM_TIMECODE_SCALE = 1000000LL` (nanoseconds); audio/video granule positions converted to this scale for Matroska
- Frame frequency must divide evenly into audio frequency: "Audio buffer size is non-integer; try lowering FPS target" ΓÇö tight coupling between game tick rate and audio sample rate

**Dual-Path Rendering Complexity:**
- OpenGL path uses FBO + glBlitFramebuffer to capture viewport at pixel_scale, handles upside-down pixel layout
- SDL surface path is simpler but less efficient (CPU-side copy)
- Conditional at every AddFrame() call (HAVE_OPENGL ifdef + MainScreenIsOpenGL() runtime check)

## Potential Issues

1. **Audio Format Assumption Fragility**: Queries OpenAL format once at Setup(), but doesn't validate it stays constant. If audio device mode changes mid-session, `in_bps` becomes stale and audio encoding corrupts.

2. **Timecode Monotonicity Enforcement**: DequeueFrames enforces `last_written_timestamp` never decreases via `prev_video_pts` / `prev_audio_packetno` checks. If a codec produces out-of-order packets, frames silently skip ΓÇö no error logged. User may not notice dropped frames in final file.

3. **Semaphore Starvation Risk**: If encode thread crashes or hangs after signaling `encodeReady` but before signaling `fillReady`, main thread blocks forever waiting for `fillReady`. No timeout or watchdog; requires external kill (unclean shutdown).

4. **File Seek Requirement**: Matroska finalization (seek head rewrite) requires seekable output. If file opened via pipe or network stream, SDL_RWseek fails silently, leaving seek head uninitialized ΓÇö file structure broken.

5. **Quality Scaling Edge Cases**: ScaleQuality() does naive linear interpolation; quality=1 gives a very different result than quality=0 due to lerp rounding. No clamping or monotonicity check; quality slider may feel non-smooth at boundaries.
