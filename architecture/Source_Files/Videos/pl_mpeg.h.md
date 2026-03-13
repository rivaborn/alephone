# Source_Files/Videos/pl_mpeg.h

## File Purpose

PL_MPEG is a single-header C library providing a complete MPEG-1 video and MPEG-1 Audio Layer II (MP2) decoder with MPEG Program Stream (PS) demuxing. It enables loading, demuxing, and frame-by-frame decoding of `.mpg` files with callbacks or direct frame retrieval.

## Core Responsibilities

- **High-level API** (`plm_*`): Unified interface combining demuxer and decoders for simplified usage
- **Data source abstraction** (`plm_buffer_*`): Ring buffer or append-only buffer supporting file, memory, and callback-driven data sources
- **MPEG-PS demuxing** (`plm_demux_*`): Parse MPEG Program Stream packets with PTS timestamps; support seeking and stream probing
- **MPEG-1 video decoding** (`plm_video_*`): Decode raw video frames into YCrCb planes; support intra/inter frames, motion compensation, IDCT
- **MP2 audio decoding** (`plm_audio_*`): Decode MPEG-1 Audio Layer II frames into interleaved or separate-channel PCM samples
- **Format conversion**: YCrCbΓåÆRGB/BGR variants with BT.601 color matrix for GPU-friendly output
- **Seeking and sync**: Frame-accurate seeking, frame-skip modes, audio lead-time management for AV sync

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `plm_t` | struct (opaque) | High-level decoder state; owns demux, video/audio buffers, callbacks, timing |
| `plm_buffer_t` | struct (opaque) | Byte buffer for data source; ring or append modes; load callback support |
| `plm_demux_t` | struct (opaque) | MPEG-PS demuxer state; tracks pack/system headers, packet stream position |
| `plm_video_t` | struct (opaque) | Video decoder; MPEG-1 sequence header, macroblock state, IDCT working memory |
| `plm_audio_t` | struct (opaque) | Audio decoder; MP2 frame headers, quantizer/scalefactor tables, synthesis state |
| `plm_frame_t` | struct (concrete) | Decoded video: time, display dimensions, 3 planes (Y, Cr, Cb); valid until next decode |
| `plm_samples_t` | struct (conditional) | Audio samples: time, count (1152), interleaved or dual-channel float arrays |
| `plm_packet_t` | struct (concrete) | Demuxed packet: type, PTS, length, raw byte data |
| `plm_plane_t` | struct (concrete) | Single color plane: width, height, raw byte data |
| `plm_quantizer_spec_t` | struct (concrete) | Audio quantizer descriptor: levels, grouping, bit depth (6 entries in LUT) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PLM_AUDIO_SAMPLE_RATE` | `uint16_t[]` (8 entries) | file-static | MPEG-1/2 sample rates (44.1, 48, 32 kHz variants) |
| `PLM_AUDIO_BIT_RATE` | `int16_t[]` (28 entries) | file-static | MPEG-1/2 bit rates for Layer II audio |
| `PLM_AUDIO_SCALEFACTOR_BASE` | `int[]` (3 entries) | file-static | Scalefactor exponent bases for audio dequantization |
| `PLM_AUDIO_SYNTHESIS_WINDOW` | `float[]` (512 entries) | file-static | Windowing function for MP2 IMDCT synthesis |
| `PLM_AUDIO_QUANT_LUT_STEP_*` | `uint8_t[][]` tables | file-static | Multi-step quantizer LUT for MP2 bit allocation |
| `PLM_AUDIO_QUANT_TAB` | `plm_quantizer_spec_t[]` (17 entries) | file-static | Quantizer specs indexed by allocation table |
| `PLM_VIDEO_ZIG_ZAG` | (not shown in excerpt) | file-static | DCT coefficient reordering for video dequantization |
| `PLM_VIDEO_PREMULTIPLIER_MATRIX` | (not shown in excerpt) | file-static | Pre-computed IDCT scale factors |

## Key Functions / Methods

### plm_create_with_filename / plm_create_with_file / plm_create_with_memory / plm_create_with_buffer
- **Signature:** `plm_t* plm_create_with_filename(const char *filename)` and variants
- **Purpose:** Constructor variants to initialize high-level decoder from file path, file handle, memory block, or custom buffer
- **Inputs:** Filename/file handle/memory pointer; optional flags for resource ownership (close_when_done, free_when_done, destroy_when_done)
- **Outputs/Return:** `plm_t*` instance or NULL on failure
- **Side effects:** Allocates demuxer, sets up video/audio buffers with load callbacks, triggers header parsing
- **Calls:** `plm_buffer_create_*`, `plm_demux_create`, `plm_init_decoders`
- **Notes:** Constructor chain: filename ΓåÆ buffer ΓåÆ demux. Early header parse attempt on init.

### plm_decode
- **Signature:** `void plm_decode(plm_t *self, double seconds)`
- **Purpose:** Advance internal timer by delta-time and decode all video/audio up to that point; invoke callbacks
- **Inputs:** Delta time since last call (in seconds)
- **Outputs/Return:** None (uses callbacks)
- **Side effects:** Updates internal time, decodes frames, calls `video_decode_callback` and `audio_decode_callback` zero or more times
- **Calls:** `plm_read_packets`, `plm_video_decode`, `plm_audio_decode`
- **Notes:** No frame-skip; decodes everything up to target time. Respects audio lead-time for AV sync.

### plm_decode_video / plm_decode_audio
- **Signature:** `plm_frame_t* plm_decode_video(plm_t *self)` / `plm_samples_t* plm_decode_audio(plm_t *self)`
- **Purpose:** Decode exactly one frame/sample block and return it; allows manual sync control
- **Inputs:** None
- **Outputs/Return:** `plm_frame_t*` or `plm_samples_t*` (valid until next decode call), or NULL on end-of-stream/error
- **Side effects:** Advances internal decoder time by 1/framerate (video) or frame duration (audio)
- **Calls:** `plm_video_decode`, `plm_audio_decode` (lower-level decoders)
- **Notes:** Caller responsible for AV sync; check `plm_has_ended()` after NULL return

### plm_seek / plm_seek_frame
- **Signature:** `int plm_seek(plm_t *self, double time, int seek_exact)` / `plm_frame_t* plm_seek_frame(plm_t *self, double time, int seek_exact)`
- **Purpose:** Jump to a target time; optionally decode to exact frame (slow) or nearest intra frame (fast)
- **Inputs:** Target time (clamped 0ΓÇôduration); seek_exact flag
- **Outputs/Return:** TRUE/FALSE (plm_seek) or frame pointer (plm_seek_frame); NULL if no frame found
- **Side effects:** Seeks demuxer, decodes video/audio up to sync point, calls decode callbacks (except plm_seek_frame)
- **Calls:** `plm_demux_seek`, `plm_video_decode`, `plm_audio_decode` in loop
- **Notes:** Only works on seekable buffers (file, fixed memory, append-mode). Exact seeking decodes all intervening frames.

### plm_video_decode
- **Signature:** `plm_frame_t* plm_video_decode(plm_video_t *self)`
- **Purpose:** Decode and return one MPEG-1 video frame
- **Inputs:** None
- **Outputs/Return:** `plm_frame_t*` with Y/Cr/Cb planes, or NULL if no data
- **Side effects:** Advances internal time by 1/framerate; updates macroblock prediction state
- **Calls:** `plm_video_decode_macroblock`, `plm_buffer_read`, bit-stream parsing helpers
- **Notes:** Handles I/P/B-frame types; motion compensation within reference frames; frame valid until next call

### plm_audio_decode
- **Signature:** `plm_samples_t* plm_audio_decode(plm_audio_t *self)`
- **Purpose:** Decode one MP2 audio frame (1152 samples)
- **Inputs:** None
- **Outputs/Return:** `plm_samples_t*` with PCM or dual-channel floats, or NULL if no data
- **Side effects:** Advances time by 1152/samplerate; updates synthesis state (V, v_pos)
- **Calls:** `plm_audio_decode_frame`, `plm_buffer_read`, `plm_audio_idct36`
- **Notes:** Frame valid until next call; conditional on PLM_AUDIO_SEPARATE_CHANNELS compile flag

### plm_frame_to_rgb / plm_frame_to_bgr / plm_frame_to_rgba / etc.
- **Signature:** `void plm_frame_to_rgb(plm_frame_t *frame, uint8_t *dest, int stride)` (6 variants)
- **Purpose:** Convert YCrCb to packed RGB/BGR variants (3 or 4 bytes per pixel)
- **Inputs:** Source frame, destination buffer, stride (bytes per row)
- **Outputs/Return:** Pixels written to dest buffer
- **Side effects:** None (read-only input, write output)
- **Calls:** None (macro expansion of PLM_DEFINE_FRAME_CONVERT_FUNCTION)
- **Notes:** BT.601 color matrix applied; macro generates all 6 variants; used for CPU-side rendering

### plm_buffer_create_with_capacity / plm_buffer_create_for_appending
- **Signature:** `plm_buffer_t* plm_buffer_create_with_capacity(size_t capacity)` / `plm_buffer_create_for_appending(size_t capacity)`
- **Purpose:** Create empty buffer for streaming data; auto-grow behavior differs (ring vs. append)
- **Inputs:** Initial capacity in bytes
- **Outputs/Return:** `plm_buffer_t*` instance
- **Side effects:** Allocates ring or append buffer; not yet seekable
- **Calls:** PLM_MALLOC
- **Notes:** Ring buffers discard read data; append buffers keep all data (seek support)

### plm_buffer_write / plm_buffer_signal_end
- **Signature:** `size_t plm_buffer_write(...)` / `void plm_buffer_signal_end(plm_buffer_t *self)`
- **Purpose:** Copy data into buffer; mark end-of-stream
- **Inputs:** Bytes to write; none (signal_end)
- **Outputs/Return:** Bytes written (always = input length unless buffer is _with_memory)
- **Side effects:** Auto-realloc on capacity exceeded; signal_end clears on rewind
- **Calls:** PLM_REALLOC
- **Notes:** Used for streaming or network-loaded files; load callback called when buffer empty

### plm_demux_decode
- **Signature:** `plm_packet_t* plm_demux_decode(plm_demux_t *self)`
- **Purpose:** Parse and return next MPEG-PS packet
- **Inputs:** None
- **Outputs/Return:** `plm_packet_t*` (type, PTS, data), or NULL if no packet
- **Side effects:** Advances bit position in buffer; may read headers on first call
- **Calls:** Buffer bit-stream readers, header parsers
- **Notes:** Packet valid until next demux_decode call; PTS may be invalid (PLM_PACKET_INVALID_TS)

## Control Flow Notes

**Initialization:**
1. User creates plm instance via `plm_create_with_*()` ΓåÆ allocates demux, attempts initial header read
2. On first decode or `plm_has_headers()` call ΓåÆ demux reads pack/system headers, initializes video/audio decoders
3. Video/audio buffers installed with load callbacks to pull packets as needed

**Frame-by-frame decoding (callback mode):**
1. App calls `plm_decode(delta_time)` repeatedly in game loop
2. PLM advances timer, demuxes video/audio packets into separate buffers
3. Video/audio decoders consume from those buffers, invoke user callbacks with decoded frames/samples
4. App renders/plays output

**Direct frame access (manual sync mode):**
1. App calls `plm_decode_video()` / `plm_decode_audio()` in alternation
2. Each returns one frame/sample block; app manages timing and AV sync
3. Typically used when app has its own audio engine

**Seeking:**
1. User calls `plm_seek(target_time, exact)` ΓåÆ demuxer jumps to nearest intra frame before target
2. If exact, video decoder replays frames from last I-frame to reach exact target
3. Audio decoder fills lead-time buffer to keep AV sync
4. Decode callbacks invoked during seek

## External Dependencies

- **Standard library:** `<stdint.h>`, `<stdio.h>`, `<string.h>`, `<stdlib.h>` (via implementation section)
- **Memory management:** `malloc`, `realloc`, `free` (replaceable via PLM_MALLOC/PLM_REALLOC/PLM_FREE macros)
- **Preprocessor:** Requires `#define PL_MPEG_IMPLEMENTATION` in one .c file before include for full implementation
- **Platform assumptions:** C99 or later; assumes IEEE 754 floats for audio; bit-shifting operations for fixed-point math
- **No external libraries:** Completely self-contained; suitable for embedded/game use
