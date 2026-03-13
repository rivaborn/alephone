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

## External Dependencies
- **SDL2**: `SDL_Thread`, `SDL_sem`, `SDL_Surface`, `SDL_Rect` for threading and surfaces
- **libav/ffmpeg**: `struct libav_vars` (opaque; implementation elsewhere)
- **OGL_FBO.h**: Frame buffer object for OpenGL capture (conditional `HAVE_OPENGL`)
- **cseries.h**: Base types, utilities, includes
- **Standard library**: `<memory>`, `<string>`, `<vector>`, `<queue>` for containers and RAII

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

## External Dependencies

- **Standard library:** `<stdint.h>`, `<stdio.h>`, `<string.h>`, `<stdlib.h>` (via implementation section)
- **Memory management:** `malloc`, `realloc`, `free` (replaceable via PLM_MALLOC/PLM_REALLOC/PLM_FREE macros)
- **Preprocessor:** Requires `#define PL_MPEG_IMPLEMENTATION` in one .c file before include for full implementation
- **Platform assumptions:** C99 or later; assumes IEEE 754 floats for audio; bit-shifting operations for fixed-point math
- **No external libraries:** Completely self-contained; suitable for embedded/game use


