# Source_Files/Videos/pl_mpeg.h - Enhanced Analysis

## Architectural Role

PL_MPEG is a self-contained, single-header MPEG-1 video/audio decoder library integrated into Aleph One's Videos subsystem for playback of legacy `.mpg` files (primarily demo recordings and cinematics from the Marathon franchise). It bridges the **Files** subsystem (which supplies raw media data via the buffer abstraction) and the **Videos/Movie** layer (which manages playback state, seeking, and frame delivery to **RenderMain** and **Sound** subsystems). By encapsulating format-specific parsing and codec logic, it isolates the engine from MPEG-1 bitstream complexity while exposing a simple object-oriented C interface aligned with the engine's 1990sΓÇô2000s architecture.

## Key Cross-References

### Incoming (who depends on this file)

- **Videos/Movie.h** (`AddFrame()`, playback controllers) ΓÇö instantiates `plm_t` decoder objects, registers video/audio callbacks, calls `plm_decode()` on each frame tick to advance internal timer and deliver decoded frames/samples to rendering and audio pipelines
- **RenderOther/screen_drawing.cpp** ΓÇö likely renders decoded video frames to screen framebuffer (via Movie playback state)
- **Sound/SoundManager / Sound/AudioPlayer** ΓÇö consumes PCM samples from `plm_samples_t` (via audio decode callback) for spatial audio mixing and output to SDL audio device

### Outgoing (what this file depends on)

- **CSeries/cstypes.h** ΓÇö `stdint.h` fixed-width integer types (`uint8_t`, `int16_t`, etc.) and byte-order operations implicit in bit-stream parsing
- **Standard C library** ΓÇö `malloc()`, `realloc()`, `free()` (replaceable via `PLM_MALLOC`/`PLM_REALLOC`/`PLM_FREE` compile macros); `stdio.h` file handle operations (`fopen`, `fread`, `fseek`)
- **Files subsystem (implicit)** ΓÇö callers construct `plm_t` with file path or `FILE*` handle, relying on the Files layer to open/manage the underlying file resource; no direct function calls into Files, but tight coupling via `FILE*` ownership semantics
- **No explicit subsystem calls** ΓÇö all decoding is self-contained; codec algorithm tables (e.g., `PLM_AUDIO_SYNTHESIS_WINDOW`, quantizer LUTs) are file-static globals embedded in the implementation section

## Design Patterns & Rationale

| Pattern | Evidence | Rationale |
|---------|----------|-----------|
| **Single-Header Library** | Entire API + implementation in one `.h` file; `#define PL_MPEG_IMPLEMENTATION` guard | Simplifies deployment (no separate `.c` file compilation); popular in game engines (stb, sokol); reduces linker complexity |
| **Opaque Type (OOP in C)** | `plm_t`, `plm_buffer_t`, `plm_demux_t`, `plm_video_t`, `plm_audio_t` are forward-declared; implementation hidden | Encapsulation: users cannot accidentally corrupt internal state; enables future refactoring without ABI breaks |
| **Object Factory Pattern** | Multiple constructor variants (`plm_create_with_filename`, `plm_create_with_file`, `plm_create_with_memory`, `plm_create_with_buffer`) | Flexibility: supports file I/O, in-memory blobs, custom buffers (streaming, network); single entry point to initialization |
| **Callback-Driven Delivery** | `plm_set_video_decode_callback()`, `plm_set_audio_decode_callback()` + `plm_decode(delta_time)` loop | Decouples codec from application's frame scheduling; app controls timing, codec does work on demand; natural fit for game loops |
| **Buffer Abstraction Layer** | `plm_buffer_*` subsystem; ring vs. append modes; load callbacks | Enables streaming (live network download), ring buffers (low-memory), or full-buffer seeks; isolates demux/codec from I/O source |
| **Deterministic Decoding** | Fixed sample counts (`PLM_AUDIO_SAMPLES_PER_FRAME = 1152`); frame-accurate seeking via intra frames | Critical for replay determinism (Aleph One supports network replays); allows precise AV synchronization without jitter |
| **Bit-Stream Parsing Macros** | Implicit in first-pass doc; MPEG bitstream accessed via buffer bit readers | Typical of codec libraries; reduces bit-level indexing boilerplate; C99 inline functions likely used under the hood |

**Tradeoff:** The single-header design prioritizes ease of integration over separate compilation (no separate `.c` file = no build speedup, full recompile each include); modern engines prefer modular shared libraries (ffmpeg, libvpx). However, plmpeg's narrow scope (MPEG-1 only) justifies the design.

## Data Flow Through This File

```
ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé                      Application Layer                       Γöé
Γöé  (Videos/Movie: playback control, frame scheduling)         Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö¼ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
                       Γöé plm_create_with_filename()
                       Γöé plm_set_video/audio_decode_callback()
                       Γöé plm_decode(delta_time) [game loop, 30 FPS]
                       Γû╝
ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé  plm_t (High-Level Orchestrator)                             Γöé
Γöé  Γö£ΓöÇ timer (internal playback time)                           Γöé
Γöé  Γö£ΓöÇ plm_demux_t (MPEG-PS packet stream)                      Γöé
Γöé  Γö£ΓöÇ plm_video_t (MPEG-1 decoder state)                       Γöé
Γöé  Γö£ΓöÇ plm_audio_t (MP2 decoder state)                          Γöé
Γöé  ΓööΓöÇ video/audio buffers, callbacks, lead-time mgmt          Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö¼ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
                       Γöé
        ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö╝ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
        Γû╝              Γû╝              Γû╝
    ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ  ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ  ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
    Γöé Buffer Γöé  Γöé Demuxer  Γöé  Γöé Video Codec Γöé
    Γöé(Ring orΓöé  Γöé(MPEG-PS) Γöé  Γöé(MPEG-1)     Γöé
    ΓöéAppend) Γöé  ΓöéPacket    Γöé  ΓöéI/P/B-frames Γöé
    Γöé        Γöé  Γöéstream    Γöé  ΓöéY/Cr/Cb 4:2:0Γöé
    ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ  ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ  ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
        Γöé           Γöé              Γöé
        Γöé           Γöé              Γû╝
        Γöé           Γöé        ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
        Γöé           Γöé        Γöé Audio Codec  Γöé
        Γöé           Γöé        Γöé(MP2 Layer II)Γöé
        Γöé           Γöé        ΓöéPCM 1152 samp Γöé
        Γöé           Γöé        ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
        Γöé           Γöé              Γöé
        ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö┤ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö┤ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
                                   Γöé
                                   Γû╝
                        ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
                        Γöé Callbacks Invoked  Γöé
                        Γöé video: plm_frame_t Γöé
                        Γöé audio: plm_samples Γöé
                        ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
                                   Γöé
                        ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö┤ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
                        Γû╝                     Γû╝
                    [RenderMain]         [Sound/Audio]
```

**Key state transitions:**
1. **Header Parse** ΓåÆ On first decode, demuxer reads MPEG pack/system headers; dimensions/framerate/samplerate locked
2. **Packet Stream** ΓåÆ Demuxer walks variable-length packets with PTS timestamps; routes video/audio to respective decoders
3. **Video Decode** ΓåÆ MPEG-1 decoder: parse sequence headers, intra/inter frames, macroblock data; perform motion compensation; output YCrCb planes
4. **Audio Decode** ΓåÆ MP2 decoder: parse frame header, bit allocation, requantize, IMDCT synthesis; output PCM samples
5. **AV Sync** ΓåÆ High-level API respects `audio_lead_time` to buffer audio ahead of video; manual sync mode (`plm_decode_video`/`plm_decode_audio`) leaves timing to caller

## Learning Notes

- **MPEG-1 Codec Archaeology:** This file encodes knowledge of a 1990s video standard (MPEG Video Committee, ISO/IEC 13818-2), now obsolete. Modern engines use H.264 (2003) or H.265/HEVC (2013). Aleph One's retention of MPEG-1 reflects its legacy media archive (Marathon demo files).
- **Deterministic Codec Design:** Frame counts and sample rates are fixed (1152 audio samples per frame, exact framerate from header). This is idiomatic to pre-GPU era codec libraries; enables frame-accurate seeking and replay reproducibility essential for netplay.
- **C99 / C11 Idioms:** Single-header library pattern, opaque type pattern, and bit-shift operations reflect early 2000s portable C practices. Modern C libraries favor `inttypes.h`, inline functions, and module separation.
- **Buffer Abstraction Lesson:** The `plm_buffer_*` subsystem demonstrates how to decouple data source (file, memory, network stream) from codec logicΓÇöa pattern reused in ffmpeg's AVIOContext. Teaches separation of concerns.
- **Callback Architecture:** Inverse-of-control pattern (codec drives app, not vice versa) common in audio/video libraries of that era (DirectShow filters, QuickTime callbacks). Modern trend favors iterator/pull APIs (codec returns frames on demand).

## Potential Issues

- **Memory Reallocation During Playback:** `plm_buffer_write()` calls `PLM_REALLOC()` if capacity exceeded. On ring buffers, this is mitigated by discarding read data; on append buffers, memory grows unbounded. In streaming scenarios (e.g., network playback), could cause frame drops or stutters if realloc blocks for too long.
- **Single-Threaded Codec:** No locks on `plm_t` state; concurrent calls to `plm_decode()` or `plm_decode_video()` from different threads will race. Aleph One's 30 FPS game loop is single-threaded, so safe in practice, but callers must externally synchronize if using async I/O or worker threads.
- **Legacy Format Robustness:** MPEG-1 parsers may not gracefully handle malformed or truncated files; the spec allows variable-length packets and optional headers. No explicit error recovery visible in first-pass; decoding might silently produce garbage or hang on corrupt headers. Recommend file validation before playback in untrusted scenarios.
- **No GPU Acceleration:** IDCT, motion compensation, color space conversion all happen on CPU. Modern real-time engines offload to GPU shader stages. For high-resolution or high-framerate MPEG-1 (rare), CPU bottleneck is inevitable.
