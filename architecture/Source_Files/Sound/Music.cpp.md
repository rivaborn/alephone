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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Music::Slot` | nested class | Manages individual music playback instances with state, fading, and OpenAL integration |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `music_slots` | `std::vector<Slot>` | member | Playback slots for intro and level music; sized to `reserved_music_slots` (2) |
| `playlist` | `std::vector<FileSpecifier>` | member | Queue of songs for level music; grows dynamically |
| `song_number` | `size_t` | member | Current index in playlist; incremented after playback or randomized |
| `random_order` | `bool` | member | If true, pick songs randomly; otherwise sequential |
| `marathon_1_song_index` | `short` | member | Cached index of classic Marathon song (for revert/reset) |
| `randomizer` | `GM_Random` | member | PRNG for shuffle mode; seeded by audio tick |

## Key Functions / Methods

### Music()
- **Signature:** `Music::Music()`
- **Purpose:** Initialize singleton instance with default state.
- **Inputs:** None.
- **Outputs/Return:** None (constructor).
- **Side effects:** Initializes `music_slots` vector with `reserved_music_slots` (2) empty slots; sets `song_number=0`, `random_order=false`, `marathon_1_song_index=NONE`.
- **Calls:** Vector construction.
- **Notes:** Uses singleton pattern; private constructor enforces single instance via `instance()` static method.

### Slot::Open
- **Signature:** `bool Music::Slot::Open(FileSpecifier* file)`
- **Purpose:** Load a single audio file and initialize it as a playable sequence with one track.
- **Inputs:** `file` ΓÇô pointer to file specification (nullable).
- **Outputs/Return:** `true` if all three steps succeed (AddTrack, AddSequence, AddSegmentToSequence); `false` otherwise.
- **Side effects:** Calls `Close()` first to reset; creates `StreamDecoder` from file; populates `dynamic_music_tracks` and `dynamic_music_sequences`.
- **Calls:** `Close()`, `AddTrack()`, `AddSequence()`, `AddSegmentToSequence()`, `StreamDecoder::Get()`.
- **Notes:** Assumes file exists and is readable; returns early if file pointer is null or decoder fails.

### Slot::Play
- **Signature:** `void Music::Slot::Play(uint32_t sequence_index=0, uint32_t segment_index=0)`
- **Purpose:** Start playback of a specific sequence and segment via OpenAL.
- **Inputs:** `sequence_index` ΓÇô target sequence (default 0); `segment_index` ΓÇô target segment (default 0).
- **Outputs/Return:** None.
- **Side effects:** Creates `musicPlayer` shared pointer via `OpenALManager::PlayMusic()`; returns silently if already playing, OpenAL unavailable, or indices invalid.
- **Calls:** `OpenALManager::Get()->PlayMusic()`, `IsSegmentIndexValid()`.
- **Notes:** Does not restart if already playing; guards against invalid indices before calling OpenAL.

### Idle
- **Signature:** `void Music::Idle()`
- **Purpose:** Frame-update function; load level music on game start, apply fading curves, and clean up finished fades.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Checks `get_game_state() == _game_in_progress` to trigger `LoadLevelMusic()` if level slot not playing; modulates volume on fading slots; may pause slots that fade to zero with `stopOnNoVolume` set.
- **Calls:** `SoundManager::IsInitialized()`, `SoundManager::IsActive()`, `OpenALManager::Get()->IsPaused()`, `get_game_state()`, `LoadLevelMusic()`, `ComputeFadingVolume()`, `SetVolume()`, `StopFade()`, `Pause()`.
- **Notes:** Gate checked first; if SoundManager offline or OpenAL paused, returns early. Iterates all slots by index to allow safe removal.

### Slot::ComputeFadingVolume
- **Signature:** `std::pair<bool, float> Music::Slot::ComputeFadingVolume() const`
- **Purpose:** Calculate interpolated volume during an active fade, applying the selected easing curve.
- **Inputs:** None (reads fade state from member variables).
- **Outputs/Return:** Pair of `(fadeIn: bool, currentVolume: float)`.
- **Side effects:** None.
- **Calls:** `SoundManager::GetCurrentAudioTick()`, `std::sin()`, `std::cos()`, `std::clamp()`.
- **Notes:** Detects fade direction from `music_fade_limit_volume > music_fade_start_volume`; uses `factor = elapsed / duration` clamped to [0,1]; handles `FadeType::Linear` (linear interpolation), `FadeType::Sinusoidal` (sin/cos easing), and `FadeType::None` (instant jump); returns exact limit at `factor==1.0` to avoid float precision drift.

### Slot::Fade
- **Signature:** `void Music::Slot::Fade(float limitVolume, short duration, MusicPlayer::FadeType fadeType, bool stopOnNoVolume=true)`
- **Purpose:** Configure a fade transition for this slot (does not compute volume, only sets parameters).
- **Inputs:** `limitVolume` ΓÇô target volume; `duration` ΓÇô fade time in ticks; `fadeType` ΓÇô easing type; `stopOnNoVolume` ΓÇô whether to pause at zero.
- **Outputs/Return:** None.
- **Side effects:** Records fade start time, target/limit volumes, duration, and type; returns early if not playing.
- **Calls:** `SoundManager::GetCurrentAudioTick()`.
- **Notes:** Must be paired with periodic calls to `ComputeFadingVolume()` and `SetVolume()` from `Idle()`; does not call `Play()` or modify `musicPlayer` directly.

### Add
- **Signature:** `std::optional<uint32_t> Music::Add(const MusicParameters& parameters, FileSpecifier* file=nullptr)`
- **Purpose:** Create and register a new music slot with optional file and parameters.
- **Inputs:** `parameters` ΓÇô volume, loop mode; `file` ΓÇô optional audio file (can be null for parameter-only add).
- **Outputs/Return:** Index of new slot if success, `std::nullopt` if file open or parameter set fails.
- **Side effects:** Pushes new `Slot` onto `music_slots` vector; modifies vector size.
- **Calls:** `Slot::Open()`, `Slot::SetParameters()`.
- **Notes:** Used to dynamically add slots beyond the two reserved; returns the index for later reference.

### LoadLevelMusic
- **Signature:** `bool Music::LoadLevelMusic()`
- **Purpose:** Fetch the next song from the playlist and open it in the level music slot.
- **Inputs:** None.
- **Outputs/Return:** `true` if slot open and parameters set; `false` otherwise.
- **Side effects:** Calls `GetLevelMusic()` to advance song index; invokes `Slot::Open()` and `Slot::SetParameters()` on `music_slots[MusicSlot::Level]`.
- **Calls:** `GetLevelMusic()`, `Slot::Open()`, `Slot::SetParameters()`.
- **Notes:** Sets loop mode to `true` only if playlist has exactly one song, allowing single-song repeats.

### GetLevelMusic
- **Signature:** `FileSpecifier* Music::GetLevelMusic()`
- **Purpose:** Select the next song from the playlist (sequentially or randomly) and advance the playlist counter.
- **Inputs:** None.
- **Outputs/Return:** Pointer to selected `FileSpecifier`, or `nullptr` if playlist empty.
- **Side effects:** Increments `song_number` for sequential mode; wraps to 0 if exceeded; picks random song if `random_order=true`; modifies `randomizer` state on random selection.
- **Calls:** `randomizer.KISS()`.
- **Notes:** Returns address of element in `playlist` vector; always advances counter after returning, so repeated calls yield different songs.

### Pause / StopLevelMusic / StopInGameMusic
- **Signature:** `void Music::Pause()`, `void Music::StopLevelMusic()`, `void Music::StopInGameMusic()`
- **Purpose:** Stop all music, stop level slot only, or stop all slots beyond intro.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** `Pause()` calls `Pause()` on all slots and resizes vector to `reserved_music_slots`; `StopLevelMusic()` closes level slot; `StopInGameMusic()` closes all slots from index `MusicSlot::Level` onward.
- **Calls:** `Slot::Pause()`, `Slot::Close()`.
- **Notes:** Cleanup functions for level/game transitions.

### SetClassicLevelMusic
- **Signature:** `void Music::SetClassicLevelMusic(short song_index)`
- **Purpose:** Load a song in the classic Marathon format (`Music/NN.ogg` or `Music/NN.mp3`) into the level playlist.
- **Inputs:** `song_index` ΓÇô numeric song ID (0-based, formatted as 2-digit).
- **Outputs/Return:** None.
- **Side effects:** Constructs file path, checks file existence, clears playlist if not empty, pushes file to playlist, records `marathon_1_song_index`.
- **Calls:** `file.SetNameWithPath()`, `file.Exists()`, `PushBackLevelMusic()`.
- **Notes:** Returns early if `playlist` already has entries (prevents mixing classic and custom playlists).

### Trivial helpers
- `RestartIntroMusic()`, `Playing()`, `SetPlaylistParameters()`, `SeedLevelMusic()`, `ClearLevelPlaylist()`, `PushBackLevelMusic()` ΓÇô manage intro playback, playlist state, and randomization. Documented in header; minimal implementation.

## Control Flow Notes
- **Init:** Constructor initializes two reserved slots.
- **Per-frame:** `Idle()` is the main update entry point, called from game loop. Detects game state, loads level music on demand, and applies active fades.
- **Shutdown:** `Pause()` or `StopLevelMusic()`/`StopInGameMusic()` halt playback and release resources.
- **Game state dependency:** Responds to `get_game_state()` returning `_game_in_progress` to trigger music loading; pauses on level exit.

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
