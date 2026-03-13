# Source_Files/Sound/AudioPlayer.cpp
## File Purpose
Implementation of `AudioPlayer`, a base class for audio playback in the Aleph One engine. Manages OpenAL source lifecycle, buffer queue operations, format tracking, and playback control.

## Core Responsibilities
- Allocate, reset, and retrieve OpenAL audio sources via `OpenALManager`
- Manage a queue of 4 buffers (8192 samples each) for streamed audio
- Track and validate audio format (sample rate, mono/stereo, bit depth)
- Implement playback flow: fill buffers ΓåÆ queue ΓåÆ play ΓåÆ unqueue processed buffers
- Synchronize audio parameters with OpenAL when properties change
- Handle rewind/stop signals via atomic flags
- Support subclasses by providing virtual hooks for data generation and parameter updates

## External Dependencies
- **OpenAL:** `<AL/al.h>`, `<AL/alext.h>` ΓÇö OpenAL core API (source/buffer management, state queries)
- **OpenALManager** ΓÇö Singleton managing device, context, source pool, master volume, listener position
- **Decoder.h** ΓÇö Audio decoding interface (included in header, used by subclasses)
- **Boost:** `boost::lockfree::spsc_queue`, `boost::unordered_map` ΓÇö Thread-safe queues and hash maps
- **Standard library:** `<array>`, `<atomic>`, `<algorithm>`, `<unordered_map>`, `<memory>` (unique_ptr)

# Source_Files/Sound/AudioPlayer.h
## File Purpose
Defines the `AudioPlayer` abstract base class for managing OpenAL audio playback in the Aleph One game engine. Coordinates audio buffer management, OpenAL source configuration, and integrates with the OpenALManager for resource allocation and prioritization. Provides lock-free synchronization for real-time audio parameter updates.

## Core Responsibilities
- Abstract interface for audio playback with OpenAL sources and buffers
- Manage buffer queue lifecycle (fill ΓåÆ queue ΓåÆ unqueue ΓåÆ refill)
- Convert audio format specifications to OpenAL format enums
- Support dynamic parameter updates (gain, pitch, position) via lock-free double-buffering
- Handle stop/rewind control signals
- Provide priority-based scheduling for OpenALManager

## External Dependencies
- **OpenAL**: `<AL/al.h>`, `<AL/alext.h>`ΓÇöaudio backend (defines `ALuint`, `AL_FORMAT_*` constants)
- **Boost**: `lockfree::spsc_queue`, `unordered_map`ΓÇölock-free concurrency and hashing
- **Local**: `Decoder.h`ΓÇöaudio stream decoding interface; `AudioFormat` enum (defined elsewhere)
- **Standard**: `<atomic>`, `<algorithm>`, `<unordered_map>`, `<memory>` (std::unique_ptr)

# Source_Files/Sound/Decoder.cpp
## File Purpose
Implements factory methods for creating audio decoders. Attempts to instantiate a libsndfile-based decoder and returns it on successful file open, or null on failure.

## Core Responsibilities
- Factory method to create StreamDecoder instances
- Factory method to create Decoder instances
- Attempt to open audio files using libsndfile
- Return decoder on success, null pointer on failure

## External Dependencies
- `Decoder.h` ΓÇö base class definitions (StreamDecoder, Decoder)
- `SndfileDecoder.h` ΓÇö concrete libsndfile-based decoder implementation
- `<memory>` ΓÇö std::unique_ptr, std::make_unique
- `FileSpecifier` ΓÇö defined elsewhere (likely FileHandler.h), represents a file path/reference

# Source_Files/Sound/Decoder.h
## File Purpose
Defines abstract interfaces for audio decoding from file streams and pre-decoded files. Provides a pluggable architecture for different audio codec implementations (WAVE, Ogg Vorbis, MP3, etc.) within the Aleph One game engine's sound system.

## Core Responsibilities
- Define abstract `StreamDecoder` interface for streaming audio decoders
- Define abstract `Decoder` interface for pre-decoded/frame-based decoders
- Provide methods for file I/O: opening, closing, reading audio data
- Provide query methods for audio properties: format, sample rate, channels, endianness, duration
- Provide seeking/positioning support for random access
- Define factory methods to instantiate appropriate concrete decoder based on file type

## External Dependencies
- `<memory>` ΓÇô `std::unique_ptr` for factory return type
- `cseries.h` ΓÇô defines `uint8`, `int32`, `uint32_t` base types
- `FileHandler.h` ΓÇô defines `FileSpecifier` class for file abstraction
- `SoundManagerEnums.h` ΓÇô defines `AudioFormat` enum (`_8_bit`, `_16_bit`, `_32_float`)

# Source_Files/Sound/Music.cpp
## File Purpose
Implements a music management system for the Aleph One game engine, handling playback of intro and level music with support for dynamic music sequences, playlist management, and fade transitions. Integrates with OpenAL for audio playback and responds to game state changes.

## Core Responsibilities
- Manage multiple music playback slots (intro at slot 0, level at slot 1)
- Load and unload music files from disk via `FileSpecifier`
- Execute volume fading with multiple easing curves (linear, sinusoidal)
- Maintain dynamic music sequences composed of tracks and segments
- Support level music playlists with sequential and random playback modes
- Detect game state transitions to auto-load level music when gameplay begins
- Coordinate music transitions between sequences during playback
- Clean up resources and stop playback on demand

## External Dependencies
- **Includes in this file:**
  - `Music.h` ΓÇô class declaration, `GM_Random`, `MusicParameters`, `MusicPlayer`, `StreamDecoder`
  - `SoundManager.h` ΓÇô `GetCurrentAudioTick()`, initialization checks
  - `interface.h` ΓÇô `get_game_state()`, game constants
  - `OpenALManager.h` ΓÇô `PlayMusic()`, pause state

- **Defined elsewhere (not in this file):**
  - `OpenALManager::Get()`, `OpenALManager::PlayMusic()` ΓÇô audio backend
  - `StreamDecoder::Get()` ΓÇô file decoding
  - `SoundManager::GetCurrentAudioTick()` ΓÇô timing reference
  - `get_game_state()` ΓÇô game state query
  - `FileSpecifier` class ΓÇô file I/O abstraction

# Source_Files/Sound/Music.h
## File Purpose
Manages intro and level music playback for the game engine. Provides a singleton interface for music control with support for dynamic sequences, segments, crossfading, and randomized playlists. Handles both classic predetermined music and dynamically composed tracks with transition edges.

## Core Responsibilities
- Provides singleton access point for global music management
- Manages two reserved music slots (intro and level) with independent playback state
- Implements fade-in/fade-out effects with customizable fade types and durations
- Supports dynamic music composition via sequences, segments, and transition rules
- Manages level music playlists with optional random shuffling
- Provides setup for classic level music by song index
- Tracks fading state and applies volume modulation during transitions

## External Dependencies
- **Random.h:** `GM_Random` struct for playlist randomization
- **MusicPlayer.h:** `MusicPlayer` class (playback engine), `MusicParameters` struct, `MusicPlayer::FadeType` enum, `MusicPlayer::Segment::Edge` (transition rules)
- **FileSpecifier:** File abstraction (defined elsewhere; used for music file references)
- **StreamDecoder:** Audio decoder interface (referenced in Slot; defined elsewhere)

# Source_Files/Sound/MusicPlayer.cpp
## File Purpose
Implementation of MusicPlayer, an audio player specializing in structured music with segment-based playback, transitions, and crossfading. Manages complex audio state transitions between music sequences/segments with configurable fade effects.

## Core Responsibilities
- Decode and buffer audio data from structured music segments
- Compute and apply fade-in/fade-out transitions with linear or sinusoidal envelopes
- Mix crossfade audio when transitioning between segments with compatible formats
- Determine next playable segment and handle automatic transitions
- Process thread-safe transition requests from main thread during playback
- Configure OpenAL source properties (gain, volume) for music playback
- Track decoder positions across segment switches, especially for same-decoder crossfades

## External Dependencies
- **AudioPlayer** ΓÇö parent class; provides audio source management, format tracking
- **StreamDecoder** ΓÇö abstract decoder interface (Position, Decode, Rate, Duration, etc.)
- **OpenALManager** ΓÇö singleton for querying master/music volume
- **OpenAL** ΓÇö `alSourcef()`, `alGetError()` for source configuration
- **Standard library** ΓÇö `<optional>`, `<cmath>`, `<algorithm>`, `<memory>`, `<vector>`, `<atomic>`, `<utility>`

# Source_Files/Sound/MusicPlayer.h
## File Purpose
MusicPlayer extends AudioPlayer to manage dynamic music playback with support for sequences of audio segments, transitions, and fade effects. It handles smooth switching between sequences with configurable crossfading and fade-in/fade-out timing, thread-safe parameter updates, and integration with OpenAL audio rendering.

## Core Responsibilities
- Manage multiple music sequences, each containing multiple audio segments (decoders)
- Process audio data with transition and fade effects (linear, sinusoidal)
- Handle asynchronous sequence transition requests with crossfading logic
- Maintain thread-safe parameter updates (volume, loop) via atomic structures
- Compute transition offsets and apply fade envelopes to audio buffers
- Override AudioPlayer hooks for OpenAL source setup and audio data generation

## External Dependencies
- **AudioPlayer.h**: Base class; provides audio thread integration, OpenAL source management, buffer queueing
- **Decoder.h** (implied): `StreamDecoder` abstract class for decoding audio formats
- **OpenAL**: `<AL/al.h>`, `<AL/alext.h>` for audio output (via base class)
- **Boost**: `boost::lockfree::spsc_queue` for thread-safe parameter updates
- **Standard Library**: `<unordered_map>`, `<vector>`, `<atomic>`, `<optional>`, `<memory>`, `<utility>`

# Source_Files/Sound/OpenALManager.cpp
## File Purpose
Implements OpenAL audio device management and playback orchestration for the Aleph One game engine. Handles device initialization, audio player lifecycle, source pooling, 3D listener updates, and SDL integration for audio mixing.

## Core Responsibilities
- OpenAL device, context, and source initialization
- Audio player (sound/music/stream) lifecycle and scheduling
- Audio source pooling with priority-based allocation
- 3D spatial audio listener updates and coordinate conversion
- Master/music volume synchronization across active players
- Audio queue processing and SDL mixer callback integration

## External Dependencies
- **OpenAL-Soft:** alcLoopbackOpenDeviceSOFT, alcCreateContext, alcMakeContextCurrent, alGenSources, alListenerfv, alGenFilters, etc.
- **SDL:** SDL_OpenAudio, SDL_PauseAudio, SDL_GetAudioStatus, SDL_LockAudio, SDL_UnlockAudio, SDL_CloseAudio
- **Classes (defined elsewhere):** SoundPlayer, MusicPlayer, StreamPlayer, AudioPlayer
- **boost::lockfree:** spsc_queue (single-producer, single-consumer)
- **Logging.h:** logError macro

# Source_Files/Sound/OpenALManager.h
## File Purpose
Central audio management system for an OpenAL-based game engine. Initializes and maintains OpenAL device/context, manages playback of sounds/music/streams, handles listener positioning for 3D audio, and maps between OpenAL and SDL audio formats.

## Core Responsibilities
- Initialize and shutdown OpenAL device and audio context
- Manage lifecycle of audio players (SoundPlayer, MusicPlayer, StreamPlayer)
- Handle pause/resume/stop operations for all active audio
- Update listener position and retrieve audio parameters (volume, frequency)
- Provide source pooling and allocation for audio playback
- Map and negotiate audio format/channel configurations between OpenAL and SDL
- Load and manage optional OpenAL extensions (spatialization, direct channel remix)
- Support loopback device for audio capture/mixing
- Support HRTF (Head-Related Transfer Function) for spatial audio

## External Dependencies
- **OpenAL Soft**: ALCdevice, ALCcontext, AL* filter functions, ALC format/channel enums
- **SDL2**: SDL_AudioSpec, SDL_AudioFormat, SDL_AudioCallback
- **Boost**: boost::lockfree::spsc_queue (lock-free single-producer single-consumer)
- **Custom headers**: MusicPlayer.h, SoundPlayer.h, StreamPlayer.h (audio player implementations); implied AudioPlayer base class, StreamDecoder, world_location3d, AtomicStructure utility
- **Math**: M_PI, std::math constants for angle conversion

# Source_Files/Sound/ReplacementSounds.cpp
## File Purpose
Implements external sound file loading and management of replacement sounds for the Aleph One engine. Provides APIs to load audio files from disk, decode them, and associate them with game sound indices via a singleton registry.

## Core Responsibilities
- Load external audio files using pluggable decoders and extract audio format metadata
- Store replacement sound options indexed by sound index and slot pair
- Unload original sounds when replacements are added
- Query replacement sound options by index
- Reset or selectively remove sound replacements with proper cleanup

## External Dependencies
- **Decoder.h** ΓÇô `Decoder::Get()`, `Decoder::Frames()`, `Decoder::BytesPerFrame()`, `Decoder::Decode()`, `Decoder::GetAudioFormat()`, `Decoder::IsStereo()`, `Decoder::IsLittleEndian()`, `Decoder::Rate()`, `Decoder::Rewind()`
- **SoundManager.h** ΓÇô `SoundManager::instance()`, `SoundManager::UnloadSound()`
- **SoundFile.h** ΓÇô `SoundData`, `SoundInfo` (base class)
- **boost/unordered_map.hpp** ΓÇô hash map container
- **FileHandler.h** (via header) ΓÇô `FileSpecifier` type

# Source_Files/Sound/ReplacementSounds.h
## File Purpose
Provides a singleton registry for managing MML-specified external sound replacements in the Aleph One game engine. Enables loading custom audio files to override built-in game sounds via index and slot identifiers.

## Core Responsibilities
- Define `ExternalSoundHeader` to encapsulate metadata and loading logic for external sound files
- Provide `SoundOptions` struct to pair file paths with sound headers
- Implement `SoundReplacements` singleton to store, retrieve, and manage the collection of sound replacements
- Support add/remove/reset operations on the replacement registry

## External Dependencies
- **`SoundFile.h`** ΓÇô provides `SoundInfo` base class, `SoundData` typedef (`std::vector<uint8>`), and `FileSpecifier`
- **`boost/unordered_map.hpp`** ΓÇô hash map container for fast lookup by `(Index, Slot)` pair
- **Standard C++ library** ΓÇô `<string>`, `<memory>` (smart pointers)

# Source_Files/Sound/SndfileDecoder.cpp
## File Purpose
Implements audio file decoding using libsndfile via SDL's RWops file abstraction. Provides a decoder bridge that translates between SDL file I/O and libsndfile's virtual I/O interface, enabling playback of various audio formats (WAV, FLAC, OGG, etc.).

## Core Responsibilities
- Implement virtual I/O callbacks (`sfd_*` functions) bridging SDL_RWops Γåö libsndfile
- Open and initialize SNDFILE handles from FileSpecifier objects
- Decode audio frames into float32 PCM buffers
- Manage playback position, seeking, and rewind operations
- Clean up file resources (SDL_RWops and SNDFILE handles)
- Provide audio format metadata (channels, sample rate, frame count)

## External Dependencies
- **Notable includes:** `"SndfileDecoder.h"`, `"sndfile.h"` (libsndfile)
- **External symbols/types:** `SNDFILE`, `SF_INFO`, `SF_VIRTUAL_IO` (libsndfile); `SDL_RWops`, `SDL_RWtell`, `SDL_RWseek`, `SDL_RWread`, `SDL_RWwrite`, `SDL_RWclose` (SDL); `FileSpecifier`, `OpenedFile` (defined elsewhere); `PlatformIsLittleEndian()`, `AudioFormat::_32_float` (inherited or platform utilities)

# Source_Files/Sound/SndfileDecoder.h
## File Purpose
Defines `SndfileDecoder`, a concrete decoder implementation that loads and decodes audio files using libsndfile. Inherits from the `Decoder` interface to support the engine's sound subsystem with format-agnostic file playback.

## Core Responsibilities
- Open and manage audio files via libsndfile library
- Decode audio frames into 32-bit float PCM buffers
- Track and report playback position and seek within files
- Provide audio metadata (sample rate, channel count, duration, frame count)
- Handle file I/O through SDL_RWops abstraction layer

## External Dependencies
- **libsndfile** (`sndfile.h`) ΓÇö decoding library; defines SNDFILE, SF_INFO
- **SDL** (implicit via `SDL_RWops`) ΓÇö cross-platform I/O abstraction
- **Decoder.h** ΓÇö parent class `Decoder` and `StreamDecoder` interface
- **FileHandler.h** (via Decoder.h) ΓÇö defines `FileSpecifier`
- **SoundManagerEnums.h** (via Decoder.h) ΓÇö defines `AudioFormat` enum
- Platform utilities (`PlatformIsLittleEndian()` ΓÇö defined elsewhere)

# Source_Files/Sound/song_definitions.h
## File Purpose
Defines the data structures and constants for music/song management in the Aleph One game engine. Contains format definitions for song segments (introduction, chorus, trailer) and a static array of song instances for the game's soundtrack.

## Core Responsibilities
- Define `sound_snippet` struct for marking playback boundaries (start/end offsets)
- Define `song_definition` struct for complete song layout with multi-segment structure
- Declare song behavior flags (`_song_automatically_loops`)
- Provide `RANDOM_COUNT` macro for variable chorus repetitions
- Initialize a static array of song definitions for the engine

## External Dependencies
- `csmisc.h` ΓÇö provides `MACHINE_TICKS_PER_SECOND` constant (1000)
- `cstypes.h` (indirect) ΓÇö for fixed-width integer types (`int16`, `int32`)

# Source_Files/Sound/sound_definitions.h
## File Purpose
Defines the data structures, constants, and static lookup tables that configure sound playback behavior in the Marathon/Aleph One game engine. It specifies how different sound categories (ambient, random, event-driven) should be attenuated by distance, obstruction, and pitch, and provides the binary format for loading sound resources from disk.

## Core Responsibilities
- Define sound behavior categories (quiet, normal, loud) and their depth-based attenuation curves
- Declare flags controlling sound restart, pitch, and obstruction behavior
- Define the binary sound file format (header and per-sound definition blocks)
- Initialize static lookup tables for ambient and random sound definitions with sound codes
- Specify sound play probabilities and pitch variation ranges per sound
- Define per-sound metadata: permutation counts, offsets, memory pointers, and playback history

## External Dependencies
- **`SoundManagerEnums.h`**: Provides `NUMBER_OF_AMBIENT_SOUND_DEFINITIONS`, `NUMBER_OF_RANDOM_SOUND_DEFINITIONS`, `NUMBER_OF_SOUND_DEFINITIONS`, `MAXIMUM_SOUND_VOLUME`, and sound code enums (`_snd_*`, `_ambient_snd_*`, `_random_snd_*`)
- **`world.h`**: Provides `WORLD_ONE` (world distance unit) for attenuation curve thresholds
- **Implicit**: `FOUR_CHARS_TO_INT()` macro (defined elsewhere, used in `SOUND_FILE_TAG`)

# Source_Files/Sound/SoundFile.cpp
## File Purpose
Implements sound file loading for the Aleph One game engine, supporting both classic Mac System 7 sound resources (M1) and the modern M2 sound format. Handles parsing audio headers, loading raw audio data, managing endianness/format conversions, and lazy-loading permutations (variants) of sounds.

## Core Responsibilities
- Parse System 7 sound headers in three variants (standard, extended, compressed)
- Load raw PCM audio data with format conversions (signedΓåÆunsigned, endianness)
- Implement M1 sound file interface (Mac resource-based audio)
- Implement M2 sound file interface (modern format with multiple sources)
- Manage sound definitions and permutations (up to 5 variants per sound)
- Cache sound headers and data to minimize re-parsing
- Handle I/O errors and stream failures gracefully

## External Dependencies
- **BStream.h**: Big-endian binary stream readers (`BIStreamBE`, `AIStreamBE`).
- **FileHandler.h**: File abstractions (`OpenedFile`, `LoadedResource`, `FileSpecifier`, `opened_file_device`).
- **SoundManagerEnums.h**: `AudioFormat` enum.
- **Logging.h**: `logWarning()` macro.
- **byte_swapping.h**: `byte_swap_memory()`.
- **boost/iostreams**: Stream buffer wrapper for memory/file sources.
- **cseries.h / cstypes.h**: Integer typedefs (`uint8`, `int16`, `_fixed`), `FOUR_CHARS_TO_INT` macro, `PlatformIsLittleEndian()`.
- **Standard library:** `<memory>`, `<vector>`, `<map>`, `<assert.h>`, `<utility>`.

# Source_Files/Sound/SoundFile.h
## File Purpose
Defines abstract and concrete interfaces for loading sound definitions and audio data from Marathon 1 and 2 sound files. Handles deserialization of sound metadata (headers, pitch, looping) and raw audio samples from both binary streams and resource forks.

## Core Responsibilities
- Define containers for sound metadata (`SoundInfo`, `SoundHeader`, `SoundDefinition`)
- Provide abstract sound file loading interface (`SoundFile`)
- Implement format-specific loaders for M1 (resource-based) and M2 (binary-based) sound files
- Parse and unpack System 7 sound headers (standard, extended, compressed)
- Load audio sample data with support for multiple permutations per sound
- Track looping parameters, pitch ranges, and playback flags

## External Dependencies

- **AStream.h**: `AIStreamBE` for deserializing big-endian data (not directly used here; BStream is preferred)
- **BStream.h**: `BIStreamBE` for big-endian binary deserialization
- **FileHandler.h**: `OpenedFile`, `OpenedResourceFile`, `LoadedResource`, `FileSpecifier` for file and resource access
- **SoundManagerEnums.h**: `AudioFormat` enum, sound codes, behavior flags
- **&lt;memory&gt;**: `std::shared_ptr`, `std::unique_ptr`
- **&lt;vector&gt;**, **&lt;map&gt;**: Container types for sound data and metadata caching

# Source_Files/Sound/SoundManager.cpp
## File Purpose
Central sound management system for the game engine, handling sound loading, playback, 3D spatialization, ambient sound management, and integration with the OpenAL audio backend. Manages lifecycle of sound players, memory budgets, and parameter calculations for dynamic sound behavior based on listener position and game state.

## Core Responsibilities
- Initialize/shutdown sound system and manage sound file lifecycle
- Load and unload sound definitions and audio data with memory constraints
- Play sounds (2D directional, 3D spatialised, ambient looping) via SoundPlayers
- Calculate sound parameters (volume, stereo panning, obstruction) based on world state
- Manage ambient sound sources and update their parameters each frame
- Handle sound replacement/patching from XML configuration (MML)
- Coordinate listener updates and sound player cleanup
- Provide volume control and sound parameter adjustment with dB scaling

## External Dependencies
- **OpenAL integration**: OpenALManager (singleton, audio device/context, source pool, playback)
- **Sound file format**: SoundFile (M1/M2 formats), SoundHeader, SoundData, SoundInfo
- **Replacements**: SoundReplacements (external audio file registry)
- **World model**: world_location3d, world_distance (listener/source positioning)
- **Configuration**: InfoTree (XML parsing for sound patches), shell_options (nosound flag)
- **Media**: Movie (audio timestamp sync during recording)
- **Memory**: FileSpecifier, LoadedResource (file I/O)

---

**Architecture Notes**:
- Singleton pattern (SoundManager, OpenALManager, SoundReplacements)
- LRU memory management with configurable budget (SoundMemoryManager)
- Lazy load strategy: sounds loaded on first play request
- Separation of concerns: SoundManager (logic) Γåö OpenALManager (audio) Γåö SoundPlayer (per-voice state)
- Flexible sound patching (XML MML) without code recompilation
- 3D spatialization integrated with game world obstruction queries

# Source_Files/Sound/SoundManager.h
## File Purpose

Central singleton manager for all audio playback in the Aleph One engine. Handles sound loading, playback, volume/pitch control, spatial audio positioning, and ambient sound management. Coordinates between the sound file system and individual SoundPlayer instances.

## Core Responsibilities

- Initialize and shutdown the audio subsystem with configurable parameters (sample rate, buffer size, master volume)
- Load/unload sounds from disk and manage in-memory sound caches
- Create and manage SoundPlayer instances for active sound playback
- Compute spatial audio parameters (stereo panning, attenuation based on listener/source position)
- Track and update listener location and orientation for 3D audio
- Manage ambient sound sources separately with dedicated channels and buffer allocation
- Provide pause/resume control via RAII Pause class
- Convert sound indices (e.g., random sound permutations) to actual playable sound indices
- Calculate pitch modifiers and obstruction effects (muffling/attenuation)

## External Dependencies

- **Includes:**
  - `cseries.h` ΓÇö platform macros, data types, SDL headers
  - `FileHandler.h` ΓÇö FileSpecifier, OpenedFile, LoadedResource
  - `SoundFile.h` ΓÇö SoundFile, SoundDefinition, SoundHeader
  - `world.h` ΓÇö world_location3d, angle, world_distance
  - `SoundPlayer.h` ΓÇö SoundPlayer class and SoundParameters
  - `<set>` ΓÇö std::set for active player tracking

- **External symbols (defined elsewhere):**
  - `world_location3d *_sound_listener_proc()` ΓÇö callback providing listener location/orientation
  - `uint16 _sound_obstructed_proc(world_location3d*, bool)` ΓÇö callback checking sound obstruction
  - `void _sound_add_ambient_sources_proc(void*, add_ambient_sound_source_proc_ptr)` ΓÇö callback for ambient source enumeration
  - `SoundMemoryManager` ΓÇö forward declaration, manages sound cache
  - Various sound accessor functions: `Sound_TerminalLogon()`, `Sound_ButtonSuccess()`, etc.
  - `parse_mml_sounds()`, `reset_mml_sounds()` ΓÇö MML sound definition parsing

# Source_Files/Sound/SoundManagerEnums.h
## File Purpose
Centralized enumeration and constant definitions for the sound manager system. Extracted from the main SoundManager header to reduce header bloat and organize sound-related type definitions and identifiers used throughout the engine.

## Core Responsibilities
- Define unique identifiers for ~200+ sound effects (weapons, ambient, entity, interactive)
- Define ambient and random sound type enumerations
- Provide audio format and channel configuration constants
- Define initialization flags for sound manager configuration (bitmask-based)
- Define sound obstruction and acoustic property flags
- Define frequency adjustment constants for Doppler and sound modulation effects

## External Dependencies
- **cstypes.h**: Provides fixed-point type (`_fixed`), fixed-point constants (`FIXED_ONE`, `FIXED_ONE_HALF`), and integer type definitions (uint8, int32, etc.)


# Source_Files/Sound/SoundPlayer.cpp
## File Purpose
Implements SoundPlayer, which manages individual sound playback via OpenAL. Handles 2D/3D spatial audio, volume/behavior transitions, rewinding, and audio format conversion (including mono-to-stereo for HRTF support).

## Core Responsibilities
- Initialize and manage sound playback parameters (pitch, volume, 3D position, obstruction flags)
- Simulate and compute sound volume based on distance and obstruction state
- Configure OpenAL audio sources for 2D panning and 3D spatial positioning
- Handle smooth parameter transitions when sound properties change during playback
- Process audio data with optional mono-to-stereo conversion for HRTF compatibility
- Manage sound rewinding (both soft and fast rewind modes)
- Apply behavior parameter tables based on obstruction/muffling conditions

## External Dependencies
- **Includes:** AudioPlayer.h, OpenALManager.h, SoundManager.h
- **OpenAL API:** alSourcef, alSourcei, alSource3f, alSource3i, alGetError (AL_* constants)
- **Defined elsewhere:** AudioPlayer (base class), OpenALManager::Get(), SoundManager::GetCurrentAudioTick(), AtomicStructure<T>, SetupALResult, SoundInfo, SoundData, LoadedResource
- **STL:** std::sqrt, std::pow, std::max, std::min, std::abs, std::copy, std::memcpy, std::tuple, std::get, std::tie
- **Math:** M_PI (from cmath via OpenALManager.h)

# Source_Files/Sound/SoundPlayer.h
## File Purpose
Defines the `SoundPlayer` class that manages individual sound playback in the Aleph One game engine. Handles 3D audio positioning, volume transitions, sound parameter updates, and rewind functionality via OpenAL backend.

## Core Responsibilities
- Manage playback lifecycle for individual sound instances (initialization, parameter updates, stopping)
- Compute sound priority based on distance and volume for playback prioritization
- Handle smooth volume and parameter transitions over time
- Support 3D positioning with distance-based attenuation and obstruction effects
- Implement soft-start (fade-in) and soft-stop (fade-out) behaviors
- Support sound rewind with fast/soft rewind variants
- Convert mono audio to stereo when needed
- Update OpenAL source parameters based on computed audio properties

## External Dependencies
- **Includes:** `AudioPlayer.h` (base class), `SoundFile.h` (SoundInfo/SoundData types), `sound_definitions.h` (sound_behavior enum, world_location3d)
- **External symbols used:**
  - `AudioPlayer` (base class with OpenAL integration, atomic buffers)
  - `SoundInfo`, `SoundData` (sound metadata and PCM data)
  - `sound_behavior` enum (behavior categorization)
  - `world_location3d` (3D position type from engine world system)
  - `AtomicStructure<T>` (lock-free queue for thread-safe parameter passing)
  - OpenAL types: `ALuint` (implicitly via AudioPlayer)

# Source_Files/Sound/SoundsPatch.cpp
## File Purpose
Implements sound patching functionality that loads replacement sound definitions and audio data from binary streams or files. Enables overriding built-in sound assets with patched versions without modifying the original sound files.

## Core Responsibilities
- Parse binary sound patch streams containing "sndc" chunks with source/index pairs
- Load sound definition headers and audio data permutations from patch data
- Manage a global collection of active sound patches indexed by (source, index)
- Provide lookup API to retrieve patched sound definitions and audio data
- Integrate with the replacement sounds system to remove old definitions during patching

## External Dependencies
- **Boost.IOStreams**: `boost::iostreams::array_source`, `boost::iostreams::stream_buffer` for memory-mapped streaming
- **BStream.h**: `BIStreamBE` (big-endian binary stream reader)
- **SoundFile.h**: `SoundDefinition`, `SoundHeader`, `SoundData`, `SoundInfo`
- **ReplacementSounds.h**: `SoundReplacements` singleton for coordinating patch application
- **FileHandler.h**: `FileSpecifier`, `OpenedFile`, `opened_file_device` for file I/O

# Source_Files/Sound/SoundsPatch.h
## File Purpose
Defines the interface for patching and overriding game sound definitions at runtime. The `SoundsPatches` class allows sounds from external sources (files or binary streams) to override base game sound definitions, enabling modding/customization without modifying core assets.

## Core Responsibilities
- Manage a collection of sound definition patches indexed by source and sound index
- Load patch data from file specifiers or binary streams (big-endian format)
- Retrieve patched `SoundDefinition` objects for a given source and sound index
- Retrieve audio sample data (`SoundData`) for a given sound definition and permutation
- Provide global patch data accessors (set/get/load functions)

## External Dependencies
- `#include <map>` ΓÇö standard library container (indexed by `(source, sound_index)` pair)
- `#include <memory>` ΓÇö for `std::shared_ptr<SoundData>`
- `"BStream.h"` ΓÇö provides `BIStreamBE` (big-endian binary input stream)
- `"SoundFile.h"` ΓÇö provides `SoundDefinition`, `SoundData`, `FileSpecifier` (defined elsewhere)
- Forward declarations: `FileHandler`, `SoundDefinitionPatch` (implementations in other translation units)

# Source_Files/Sound/StreamPlayer.cpp
## File Purpose
Implements `StreamPlayer`, a callback-driven audio player component for the Aleph One game engine. Audio data is fetched on-demand from an external callback function rather than stored in memory, enabling streaming of audio sources like intro videos.

## Core Responsibilities
- Construct StreamPlayer with callback function and playback parameters
- Override `GetNextData()` to pull audio frames from a registered callback
- Manage callback function pointer and user-defined context data
- Enforce callback invocation pattern with buffer size constraints

## External Dependencies
- `AudioPlayer` (parent class via StreamPlayer.h)
- `std::min()` (standard library)

# Source_Files/Sound/StreamPlayer.h
## File Purpose
Defines `StreamPlayer`, a subclass of `AudioPlayer` that implements streaming audio playback via user-supplied callback functions. Designed for exclusive use by `OpenALManager` to play dynamic audio sources (noted for intro video playback).

## Core Responsibilities
- Implement callback-driven audio data provisioning
- Override `GetNextData()` to fetch audio samples from user callback
- Store and manage callback function pointer and opaque user data
- Declare fixed priority level for audio scheduling
- Integrate with OpenAL audio system through `AudioPlayer` base class

## External Dependencies
- `AudioPlayer.h` ΓÇö Base class providing OpenAL source management, atomic buffer queuing, and format mapping
- OpenAL headers (`AL/al.h`, `AL/alext.h`) ΓÇö Via `AudioPlayer`
- `Decoder.h` ΓÇö Provides `AudioFormat` enum (via `AudioPlayer`)
- `boost/lockfree/spsc_queue.hpp`, `boost/unordered_map.hpp` ΓÇö Via `AudioPlayer` for lock-free structures and hash maps


