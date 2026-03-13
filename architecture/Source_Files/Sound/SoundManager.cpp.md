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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| SoundMemoryManager | class | LRU-based memory pool for loaded sound data; evicts oldest-played sounds when budget exceeded |
| SoundMemoryManager::Entry | struct | Per-sound metadata: vector of SoundData pointers (up to 5 permutations), last_played timestamp |
| SoundManager::Parameters | struct | Configuration: volume_db, flags (3d/ambient/16bit), sample rate, channel layout |
| SoundManager::SoundVolumes | struct | Runtime volume levels: global, left_volume, right_volume |
| ambient_sound_data | struct | Active ambient sound: flags, sound_index, SoundVolumes for accumulation |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| _Sound_TerminalLogon, etc. (20 static shorts) | short | file-static | M1-compatibility: indices of formerly hardcoded UI/event sounds |
| original_ambient_sound_definitions | ambient_sound_definition* | file-static | Backup of ambient defs for MML reset |
| original_random_sound_definitions | random_sound_definition* | file-static | Backup of random defs for MML reset |
| original_dialog_sound_definitions | int16* | file-static | Backup of dialog defs for MML reset |

## Key Functions / Methods

### Initialize
- Signature: `void Initialize(const Parameters& new_parameters)`
- Purpose: Load initial sound file, register shutdown hook, enable sound system
- Inputs: Parameters struct with flags and settings
- Outputs/Return: None; sets `initialized=true`
- Side effects: Opens sound file, registers atexit handler, allocates SoundMemoryManager
- Calls: OpenSoundFile(), SetParameters()
- Notes: Only succeeds if sound file loads; idempotent after first call

### SetParameters
- Signature: `void SetParameters(const Parameters& parameters)`
- Purpose: Update runtime sound configuration (volume, flags)
- Inputs: Parameters struct
- Outputs/Return: None
- Side effects: Validates volume bounds, calls SetStatus() to reinit OpenAL if needed
- Calls: Parameters::Verify(), SetStatus()

### LoadSound
- Signature: `bool LoadSound(short sound_index)`
- Purpose: Load all permutations of a sound definition into memory, respecting ambient/16-bit flags
- Inputs: Sound index
- Outputs/Return: true if sound loaded and available
- Side effects: Allocates sound memory, updates SoundMemoryManager, triggers eviction if budget exceeded
- Calls: GetSoundDefinition(), sounds_patches.get_sound_data(), SoundReplacements::GetSoundOptions()
- Notes: Checks cache first (IsLoaded); tries patches, then sound file, then external replacements

### PlaySound (short overload)
- Signature: `std::shared_ptr<SoundPlayer> PlaySound(short sound_index, world_location3d *source, short identifier, _fixed pitch, bool soft_rewind)`
- Purpose: Queue a sound for playback with world positioning and listener-relative audio
- Inputs: sound_index, source location (nullptr for 2D), identifier for reuse, pitch modifier, soft rewind flag
- Outputs/Return: Shared pointer to active SoundPlayer
- Side effects: Loads sound, calculates 3D parameters or stereo panning, updates OpenALManager
- Calls: LoadSound(), CalculateInitialSoundVariables(), GetSoundObstructionFlags(), BufferSound()
- Notes: Returns null if sound inactive or master volume is silent; identifier=NONE orphans sound immediately

### PlaySound (resource overload)
- Signature: `std::shared_ptr<SoundPlayer> PlaySound(LoadedResource& rsrc, const SoundParameters& parameters)`
- Purpose: Direct playback of pre-loaded sound data (for external/streaming sources)
- Inputs: LoadedResource (raw audio), SoundParameters struct
- Outputs/Return: Active SoundPlayer or null
- Side effects: Parses header, delegates to ManageSound()
- Calls: SoundHeader::Load(), SoundHeader::LoadData(), ManageSound()

### DirectPlaySound
- Signature: `std::shared_ptr<SoundPlayer> DirectPlaySound(short sound_index, angle direction, short volume, _fixed pitch)`
- Purpose: Play sound with explicit direction and volume, no listener offset calculation
- Inputs: sound_index, bearing angle from listener, volume (0ΓÇô256), pitch modifier
- Outputs/Return: SoundPlayer or null
- Side effects: Calculates stereo split based on angle, queries listener location
- Calls: LoadSound(), AngleAndVolumeToStereoVolume(), BufferSound()
- Notes: Used for UI and immediate-feedback sounds

### ManagePlayers
- Signature: `void ManagePlayers()`
- Purpose: Update parameters of all active sound players each frame (dynamic tracking, obstruction, panning)
- Inputs: None (uses sound_players list)
- Outputs/Return: None
- Side effects: Calls UpdateParameters() on players with changed location/obstruction/panning
- Calls: CleanInactivePlayers(), GetSoundObstructionFlags(), CalculateInitialSoundVariables()
- Notes: Runs in Idle(); respects dynamic_source_location3d pointer for moving sources

### GetSoundObstructionFlags
- Signature: `uint16 GetSoundObstructionFlags(short sound_index, world_location3d* source)`
- Purpose: Query obstruction state (walls, media) and apply sound definition filters
- Inputs: sound_index, source location
- Outputs/Return: Bitmask of obstruction flags (_sound_was_obstructed, _sound_was_media_obstructed, etc.)
- Side effects: None (read-only query)
- Calls: GetSoundDefinition(), _sound_obstructed_proc()
- Notes: Respects definition flags (e.g., _sound_cannot_be_obstructed)

### UpdateAmbientSoundSources
- Signature: `void UpdateAmbientSoundSources()`
- Purpose: Accumulate active ambient sound sources, cull by priority, update/start looping players
- Inputs: None (calls _sound_add_ambient_sources_proc)
- Outputs/Return: None
- Side effects: Modifies ambient_sound_players set, queries and updates SoundPlayer parameters
- Calls: CleanInactivePlayers(), _sound_add_ambient_sources_proc(), AddOneAmbientSoundSource(), LoadSound(), BufferSound()
- Notes: Culls weakest sources if count exceeds MAXIMUM_AMBIENT_SOUND_CHANNELS; soft-stops sounds no longer in list

### GetCurrentAudioTick
- Signature: `static uint64_t GetCurrentAudioTick()`
- Purpose: Get synchronized audio timestamp accounting for pause state and movie recording
- Inputs: None
- Outputs/Return: Tick count in audio thread time
- Side effects: None
- Calls: Movie::instance()->IsRecording(), Movie::instance()->GetCurrentAudioTimeStamp(), OpenALManager::Get()->GetElapsedPauseTime()
- Notes: Static; used for consistency between game logic and audio

### parse_mml_sounds / reset_mml_sounds
- Signature: `void parse_mml_sounds(const InfoTree& root)` / `void reset_mml_sounds()`
- Purpose: Load/reset sound definitions and external sound replacements from XML
- Inputs: InfoTree (XML doc) or none (reset)
- Outputs/Return: None
- Side effects: Modifies global ambient/random/dialog sound definitions, SoundReplacements registry
- Calls: InfoTree::read_attr(), root.children_named(), SoundReplacements operations
- Notes: Backs up originals on first parse; reset clears MML patches

## Notes (Utility Functions/Helpers)
- **CalculatePitchModifier** ΓÇô Applies resistance/disabled pitch change flags to pitch modifier
- **AngleAndVolumeToStereoVolume** ΓÇô Splits volume into left/right based on bearing angle (quad sectors)
- **CalculateSoundVariables / CalculateInitialSoundVariables** ΓÇô Compute volume/panning from distance and angle
- **distance_to_volume** ΓÇô Maps distance to volume using depth curves (obstructed vs. unobstructed)
- **AddOneAmbientSoundSource** ΓÇô Accumulates one ambient source's contribution to the mix
- **GetSoundPlayer** ΓÇô Find active player matching identifier/source, with priority eviction for new sources
- **BufferSound** ΓÇô Load permutation header and data, delegate to ManageSound
- **ManageSound** ΓÇô Decide to rewind existing player or create new one; update/add to sound_players
- **UpdateExistingPlayer** ΓÇô Check if existing player should be rewound instead of creating new one
- **CleanInactivePlayers** ΓÇô Remove finished players from active list

## Control Flow Notes
- **Init**: `Initialize()` ΓåÆ `OpenSoundFile()` ΓåÆ `SetStatus(true)` ΓåÆ audio system ready
- **Per-frame**: `Idle()` ΓåÆ `UpdateListener()` ΓåÆ `CauseAmbientSoundSourceUpdate()` ΓåÆ `ManagePlayers()` ΓåÆ OpenAL updated
- **Sound playback**: `PlaySound()` / `DirectPlaySound()` ΓåÆ `LoadSound()` + param calc ΓåÆ `BufferSound()` ΓåÆ `ManageSound()` ΓåÆ new SoundPlayer via OpenALManager or reuse existing
- **Ambient loop**: `UpdateAmbientSoundSources()` called from Idle; accumulates sources, culls by priority, updates looping players
- **Shutdown**: `Shutdown()` ΓåÆ `SetStatus(false)` ΓåÆ atexit cleanup

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
