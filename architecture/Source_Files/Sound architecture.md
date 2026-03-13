# Subsystem Overview

## Purpose

The Sound subsystem provides comprehensive audio playback for the Aleph One game engine, managing sound effects, music sequences, and 3D spatial audio through an OpenAL backend. It handles audio file decoding (WAV, FLAC, OGG via libsndfile), playback prioritization, real-time parameter updates, and listener-based attenuation. The subsystem coordinates between OpenAL audio sources, the game world state, and the SoundManager for centralized control of all audio operations.

## Key Files

| File | Role |
|------|------|
| OpenALManager.cpp/h | Device/context initialization, audio player lifecycle, source pooling, 3D listener updates |
| SoundManager.cpp/h | Central sound effect management, playback, spatial parameter calculation, ambient sound coordination |
| SoundPlayer.cpp/h | Per-instance sound playback with 3D positioning, volume transitions, rewind handling |
| MusicPlayer.cpp/h | Music playback with segment-based composition, crossfading, fade effects, sequence transitions |
| Music.cpp/h | Music slot management (intro/level), playlist handling, fade orchestration, game state tracking |
| AudioPlayer.cpp/h | Base class for OpenAL source lifecycle, buffer queueing, format tracking, thread-safe parameter updates |
| StreamPlayer.cpp/h | Callback-driven streaming audio for dynamic sources (intro videos) |
| Decoder.cpp/h | Abstract decoder interface for pluggable audio format support |
| SndfileDecoder.cpp/h | Concrete libsndfile-based decoder with SDL_RWops file abstraction |
| SoundFile.cpp/h | M1/M2 sound file parsing, System 7 header handling, permutation loading |
| SoundManagerEnums.h | Sound constants, audio format enums, initialization flags, ~200+ sound IDs |
| sound_definitions.h | Sound behavior categories, attenuation curves, playback flags, static lookup tables |
| song_definitions.h | Song segment structures (intro/chorus/trailer), song array definitions |
| ReplacementSounds.cpp/h | External sound file loading and override registry |
| SoundsPatch.cpp/h | Binary sound patch loading and patched sound definition lookup |

## Core Responsibilities

- Initialize/shutdown OpenAL device, context, and maintain a pooled set of audio sources with priority-based allocation
- Load audio files via pluggable decoders (libsndfile), parse M1/M2 sound file formats, and manage in-memory sound caches with configurable memory budgets
- Play individual sound effects with 3D spatial positioning, distance-based attenuation, obstruction/muffling effects, and smooth parameter transitions
- Manage music playback via MusicPlayer instances organized into sequences with support for segment composition, crossfading, and fade-in/fade-out transitions
- Calculate dynamic audio parameters (volume, stereo panning, pitch, obstruction flags) based on listener/source position and world state
- Update 3D listener position each frame and manage ambient sound sources with per-frame parameter recalculation
- Support sound replacement/patching from external audio files (MML configuration) and binary patch streams without code recompilation
- Coordinate thread-safe parameter updates between game thread and audio callback thread using lock-free data structures (boost::lockfree::spsc_queue)

## Key Interfaces & Data Flow

**Exposes to others:**
- `SoundManager` singleton: `PlaySound()`, `UnloadSound()`, volume/listener control, pitch adjustments, pause/resume, ambient sound registration
- `Music` singleton: `SetupLevelMusic()`, `PlayIntroMusic()`, fade operations, sequence management
- `OpenALManager` singleton: `PlayMusic()`, `PlaySound()`, listener updates, audio source allocation
- Audio players: `SoundPlayer`, `MusicPlayer`, `StreamPlayer` instances created on-demand for active playback

**Consumes from:**
- **World subsystem:** listener position/orientation via `_sound_listener_proc()` callback; obstruction queries via `_sound_obstructed_proc()` callback; ambient sound enumeration via `_sound_add_ambient_sources_proc()` callback
- **Files subsystem:** `FileSpecifier` for sound/music file references; `OpenedFile` and `LoadedResource` for file I/O; resource fork emulation for Mac resource access
- **XML/MML subsystem:** `parse_mml_sounds()` and `reset_mml_sounds()` for sound patch definitions from configuration
- **Rendering/Game state:** game state queries via `get_game_state()` to trigger level music transitions; audio timestamp sync via `Movie` class during recording

## Runtime Role

- **Initialization:** OpenAL device and context creation via `alcLoopbackOpenDeviceSOFT`/`alcCreateContext`; source pool allocation; SDL audio callback registration; sound file format negotiation between OpenAL and SDL
- **Per-frame:** Update 3D listener position from world callback; recalculate ambient sound parameters and requeue updates; detect game state transitions for music changes; update listener/source distance attenuation based on obstruction state
- **Per-audio-callback:** OpenAL invokes SDL mixer callback; audio players generate frames via `GetNextData()` hooks (SoundPlayer fetches decoded buffers, MusicPlayer applies fades/transitions, StreamPlayer invokes external callbacks); audio data written to OpenAL buffer queue; exhausted buffers dequeued and refilled
- **Shutdown:** Stop all active players; close OpenAL sources; destroy OpenAL context and device; release loaded sound data and decoder resources

## Notable Implementation Details

- Lock-free single-producer, single-consumer queues (`boost::lockfree::spsc_queue`) for thread-safe atomic parameter updates (volume, pitch, position) from game thread to audio callback thread without locks
- Memory budget management via `SoundMemoryManager` using LRU eviction for in-memory sound cache to constrain total audio data RAM
- Lazy-load strategy: sounds decoded and loaded into memory only on first playback request; unloaded when replaced or memory budget exceeded
- Mono-to-stereo conversion in `SoundPlayer` to enable HRTF (Head-Related Transfer Function) spatial audio processing via OpenAL extensions
- Crossfading logic in `MusicPlayer` for smooth transitions between music segments with compatible audio formats; fade envelopes computed per-buffer with linear or sinusoidal easing curves
- Dynamic music composition: sequences are containers of segments; transitions between segments encoded as directed edges with fade parameters; thread-safe transition requests processed asynchronously during playback
- Dual-format support: M1 format loads from System 7 resource forks (via `LoadedResource`); M2 format parses binary big-endian streams with `BIStreamBE`; both support permutations (up to 5 variants per sound)
- Sound patching system allows external audio files to override built-in definitions indexed by (source, sound_index) pair; patched sounds queried first, fall back to built-in sounds if no patch exists
