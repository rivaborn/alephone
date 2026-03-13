# Subsystem Overview

## Purpose

The Videos subsystem provides gameplay recording and video playback capabilities for Aleph One. It encodes game footage and audio into WebM files via background thread processing and decodes MPEG-1 video streams for in-game video playback, supporting both hardware-accelerated (OpenGL) and software rendering paths.

## Key Files

| File | Role |
|------|------|
| Movie.h | Singleton interface for recording state, frame queueing, and background thread coordination |
| Movie.cpp | WebM encoding implementation with VP8 video and Vorbis audio codecs; thread-safe producerΓÇôconsumer architecture |
| pl_mpeg.h | Self-contained MPEG-1 video and MP2 audio decoder with Program Stream demuxing and frame-accurate seeking |

## Core Responsibilities

- Capture video frames from OpenGL framebuffer or SDL software surfaces
- Encode video frames to VP8 codec with configurable bitrate and quality scaling by resolution
- Encode audio to Vorbis codec with configurable quality via OpenALManager loopback capture
- Multiplex audio and video into WebM/Matroska container with frame synchronization and cluster-based organization
- Manage thread-safe producerΓÇôconsumer pipeline: main thread captures, encode thread processes asynchronously via semaphores
- Decode MPEG-1 video frames and MP2 audio streams from Program Stream packets with PTS timestamps
- Convert between color spaces (ARGBΓåöI420 YUV, YCrCbΓåöRGB variants) for encoding and display
- Support frame-accurate seeking, frame-skip modes, and audio lead-time management for A/V synchronization

## Key Interfaces & Data Flow

**Exposes to other subsystems:**
- Movie singleton methods: start/stop recording, queue frame/audio, manage recording state
- User prompts for recording parameters and file dialogs
- MPEG playback API (`plm_*` functions) for frame-by-frame decoding and seeking

**Consumes from other subsystems:**
- **Screen**: viewport info, surface access, OpenGL detection state
- **OpenALManager**: audio device loopback capture for recording
- **RenderMain/RenderOther**: rendered frame buffers (OpenGL or SDL surfaces)
- **interface.h**: FPS target, input state, user alerts, file dialogs
- **alephversion.h**: version metadata for container

**External libraries:**
- libvpx (VP8 video), libvorbis/libogg (Vorbis audio), libmatroska/libebml (WebM container)
- libyuv (color space conversion), SDL2 (threading, surfaces, file I/O), OpenGL (FBO capture)

## Runtime Role

**Recording frame flow:**
1. Main thread captures game frame (OpenGL FBO or SDL surface) ΓåÆ queues to buffer
2. Encode thread dequeues asynchronously, encodes to VP8, writes to WebM cluster
3. Audio captured from OpenAL loopback ΓåÆ encoded to Vorbis and multiplexed with video
4. Semaphores synchronize producerΓÇôconsumer to prevent queue overflow

**Playback frame flow:**
1. MPEG Program Stream demuxer parses packets and extracts PTS timestamps
2. Video decoder converts frames to YCrCb planes; audio decoder produces PCM samples
3. Caller retrieves decoded frames via `plm_*` API or applies format conversion

## Notable Implementation Details

- **Dual capture paths**: OpenGL FBO for GPU rendering, SDL surface blits for software rendering
- **Quality scaling**: Bitrate presets derived from resolution (YouTube-like heuristics)
- **Data source abstraction** (pl_mpeg): Ring or append-only buffers support file, memory, and callback-driven inputs; enables streaming
- **Self-contained decoder**: pl_mpeg requires only standard C library (`malloc`, `realloc`, `free`); activated via `#define PL_MPEG_IMPLEMENTATION`
- **A/V synchronization**: Cluster-based multiplexing aligns video/audio frames; seeking respects audio lead-time for lip-sync
- **Format conversions**: YCrCbΓåöRGB via BT.601 color matrix; ARGBΓåöI420 YUV via libyuv for encoder input
