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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| MusicSlot | enum | Slot identifiers: Intro (0), Level (1) |
| Slot | inner class | Encapsulates a single music slot with playback, tracks, sequences, and fade state |
| MusicParameters | struct (from MusicPlayer.h) | Volume and loop settings |
| GM_Random | struct (from Random.h) | Marsaglia-based PRNG for playlist shuffling |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| m_instance | Music* | static (via instance() method) | Singleton instance pointer |
| music_slots | std::vector\<Slot\> | instance member | Array of music slots (Intro, Level) |
| playlist | std::vector\<FileSpecifier\> | instance member | Level music file queue |
| song_number | size_t | instance member | Current position in playlist |
| random_order | bool | instance member | Whether playlist is randomized |
| marathon_1_song_index | short | instance member | Classic level music index |
| randomizer | GM_Random | instance member | RNG for playlist shuffling |

## Key Functions / Methods

### instance
- **Signature:** `static Music *instance()`
- **Purpose:** Singleton accessor; creates instance on first call
- **Inputs:** None
- **Outputs/Return:** Pointer to global Music instance
- **Side effects:** Allocates Music on first call
- **Calls:** None (constructor called implicitly)

### Slot::Fade
- **Signature:** `void Fade(float limitVolume, short duration, MusicPlayer::FadeType fadeType, bool stopOnNoVolume = true)`
- **Purpose:** Initiate fade operation on this slot
- **Inputs:** Target volume limit, duration (milliseconds?), fade curve type, whether to stop playback on silence
- **Outputs/Return:** None
- **Side effects:** Sets `music_fade_start`, `music_fade_duration`, `music_fade_type`, etc.
- **Calls:** None visible
- **Notes:** Actual fading is computed lazily via `ComputeFadingVolume()`

### Slot::Open
- **Signature:** `bool Open(FileSpecifier* file)`
- **Purpose:** Load and initialize a music file into this slot
- **Inputs:** FileSpecifier pointer (file to load)
- **Outputs/Return:** Success flag
- **Side effects:** Creates MusicPlayer, populates dynamic_music_tracks/sequences
- **Calls:** Implementation not visible (likely in .cpp)

### Slot::Play
- **Signature:** `void Play(uint32_t sequence_index = 0, uint32_t segment_index = 0)`
- **Purpose:** Start playback of a sequence and segment
- **Inputs:** Sequence index (default 0), segment index within sequence (default 0)
- **Outputs/Return:** None
- **Side effects:** Initiates playback in musicPlayer
- **Calls:** Implementation not visible

### Slot::AddTrack
- **Signature:** `std::optional<uint32_t> AddTrack(FileSpecifier* file)`
- **Purpose:** Add a dynamic music track for this slot
- **Inputs:** FileSpecifier pointer
- **Outputs/Return:** Track index if successful, std::nullopt otherwise
- **Side effects:** Appends to dynamic_music_tracks
- **Calls:** Implementation not visible

### Slot::AddSequence
- **Signature:** `std::optional<uint32_t> AddSequence()`
- **Purpose:** Create a new sequence container
- **Inputs:** None
- **Outputs/Return:** Sequence index if successful
- **Side effects:** Appends to dynamic_music_sequences
- **Calls:** Implementation not visible

### Music::Add
- **Signature:** `std::optional<uint32_t> Add(const MusicParameters& parameters, FileSpecifier* file = nullptr)`
- **Purpose:** Add new music slot dynamically with custom parameters
- **Inputs:** Volume/loop parameters, optional file to load
- **Outputs/Return:** Slot index if successful
- **Side effects:** Grows music_slots vector
- **Calls:** Implementation not visible

### Music::SetupIntroMusic
- **Signature:** `bool SetupIntroMusic(FileSpecifier& file)`
- **Purpose:** Load intro music into the Intro slot
- **Inputs:** FileSpecifier reference
- **Outputs/Return:** Success flag
- **Side effects:** Initializes Intro slot (reserved index 0)
- **Calls:** `music_slots[MusicSlot::Intro].Open(&file)`

### Music::PushBackLevelMusic
- **Signature:** `void PushBackLevelMusic(const FileSpecifier& file)`
- **Purpose:** Add file to level music playlist queue
- **Inputs:** FileSpecifier by reference
- **Outputs/Return:** None
- **Side effects:** Appends to playlist vector
- **Calls:** Implementation not visible

### Music::SeedLevelMusic
- **Signature:** `void SeedLevelMusic()`
- **Purpose:** Initialize randomizer for level music playback
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Resets randomizer state
- **Calls:** Implementation not visible (likely calls `randomizer.SetTable()`)

### Music::Idle
- **Signature:** `void Idle()`
- **Purpose:** Update music state per frame
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Processes fade transitions, loads next level track, updates all slot states
- **Calls:** Implementation not visible (likely calls `ComputeFadingVolume()`, `LoadLevelMusic()`)
- **Notes:** Called each frame to drive dynamic music updates

## Control Flow Notes
- **Initialization:** `SetupIntroMusic()` is called during startup; `LoadLevelMusic()` loads next track from playlist during level playback
- **Frame update:** `Idle()` is called per frame to update fades, manage playlist progression, and handle segment/sequence transitions
- **Shutdown:** `Close()` and `StopLevelMusic()` / `StopInGameMusic()` clean up slots
- **Level-to-level:** `ClearLevelPlaylist()` and `PushBackLevelMusic()` build new playlists; `SetClassicLevelMusic()` selects predetermined tracks
- **Dynamic music:** Sequences and segments with transition edges enable adaptive music composition within a slot

## External Dependencies
- **Random.h:** `GM_Random` struct for playlist randomization
- **MusicPlayer.h:** `MusicPlayer` class (playback engine), `MusicParameters` struct, `MusicPlayer::FadeType` enum, `MusicPlayer::Segment::Edge` (transition rules)
- **FileSpecifier:** File abstraction (defined elsewhere; used for music file references)
- **StreamDecoder:** Audio decoder interface (referenced in Slot; defined elsewhere)
