# Source_Files/Sound/SoundManager.h - Enhanced Analysis

## Architectural Role

SoundManager is the **spatial audio bridge** between the 3D game world and the audio output device. It translates world geometry and entity positions (via GameWorld callbacks) into real-time 3D audio parametersΓÇöstereo panning, volume attenuation, obstruction mufflingΓÇöand manages concurrent SoundPlayer instances. This file demonstrates tight coupling to GameWorld's physics callbacks (`_sound_listener_proc`, `_sound_obstructed_proc`) and loose coupling to the rendering and network subsystems via MML configuration.

## Key Cross-References

### Incoming (who depends on this file)

- **GameWorld (map.cpp)**: Calls `_sound_obstructed_proc()` callback (implemented in SoundManager indirectly) to determine if sounds are blocked by world geometry; triggers ambient sound updates via `CauseAmbientSoundSourceUpdate()`
- **GameWorld (effects.cpp, devices.cpp)**: Calls `PlaySound()` with `world_location3d *source` to emit spatial sounds (explosions, terminal beeps, platform activation)
- **Rendering (RenderOther)**: May trigger ambient sound updates synchronized with visual effects
- **Misc/Interface**: Uses `AdjustVolumeUp/Down()` for volume control UI; creates `Pause` instances during cutscenes/menus
- **Lua scripting**: Likely accesses sound functions via engine bindings (not explicitly shown but referenced in other files)
- **Network multiplayer**: May sync sound playback across peers (inferred from sound_index identifiers allowing remote sound disambiguation)

### Outgoing (what this file depends on)

- **Files/FileHandler**: Opens `.snd` files via `OpenSoundFile(FileSpecifier&)` and loads resources with `LoadedResource`
- **Sound/SoundFile**: Manages `.snd` archive format (sound definitions, headers, metadata)
- **Sound/SoundMemoryManager**: Caches decompressed/decoded audio in RAM with size limits (300KBΓÇô1MB)
- **Sound/SoundPlayer**: Wraps individual audio streams; shared_ptr instances track playback state, position, volume
- **GameWorld (world.h)**: Imports `world_location3d` (3D position), `angle` (direction), world distance types; calls external callbacks to get listener position and check obstruction
- **CSeries**: Uses fixed-point math (`_fixed`), dB conversion utilities, platform types

## Design Patterns & Rationale

**Observer Callbacks (Inversion of Control)**  
Rather than importing GameWorld directly, SoundManager receives world state via callback pointers (`_sound_listener_proc`, `_sound_obstructed_proc`, `_sound_add_ambient_sources_proc`). This decouples the audio system from world state and allows the main loop to inject listener position and obstruction logic without SoundManager needing to know about map geometry. This is a classic callback pattern for real-time graphics/audio engines.

**Shared Pointer Pooling**  
Active SoundPlayers are tracked in `std::set<std::shared_ptr<SoundPlayer>>` and cleaned up in `Idle()` via `CleanInactivePlayers()`. This provides automatic lifecycle management without manual `new`/`delete` and allows multiple owners of a playing sound (e.g., world entity + UI pause mechanism).

**Dual-Channel Audio Architecture**  
`sound_players` and `ambient_sound_players` are managed separately: regular sounds share a common volume slider and can be directly stopped by identifier; ambient sounds have dedicated channels (4 max) and separate buffer allocation (1 MB). This reflects the design that ambient sounds are environment-driven (water flows, distant machinery) while entity sounds are event-driven.

**dB-Based Volume with Thresholds**  
Master volume is stored in dB (`-40.0` to `0.0`) rather than linear 0ΓÇô1 range, with separate thresholds for music vs. SFX and a conversion function `From_db()`. This matches professional audio engineering practice where human perception of loudness is logarithmic; a `-20 dB` change is perceived as half as loud regardless of baseline volume.

**Why This Structure?**  
The separation into SoundManager (orchestrator), SoundPlayer (stream holder), SoundFile (archive), and SoundMemoryManager (cache) follows a pipeline design: definition lookup ΓåÆ memory loading ΓåÆ playback stream creation. Each layer is testable independently and can be swapped (e.g., different SoundFile format, in-memory vs. streaming SoundPlayer).

## Data Flow Through This File

```
World Event (explosion at location X)
    Γåô
GameWorld calls PlaySound(sound_index, world_location3d *source, ...)
    Γåô
SoundManager::GetSoundDefinition(sound_index)  [lookup in SoundFile]
    Γåô
GetRandomSoundPermutation(sound_index)  [pick variant if multishot]
    Γåô
GetSoundObstructionFlags(source)  [call _sound_obstructed_proc]
    Γåô
CalculateSoundVariables(source)  [compute stereo panning, volume attenuation from listener distance]
    Γåô
BufferSound(SoundParameters)  [request SoundMemoryManager to decompress into RAM]
    Γåô
SoundPlayer::PlaySound(buffer, pitch, volume)  [begin playback]
    Γåô
Per-frame UpdateListener()  [listener moves, recalculate all active sound pannings]
    Γåô
Idle()  [mark finished sounds for removal, clean up shared_ptr set]
```

Key state transitions:
- **Load**: Sound loaded into cache; marked as "in use" until `UnloadSound()` or cache eviction
- **Playing**: SoundPlayer active in set; volume/pan recomputed each frame
- **Finishing**: SoundPlayer duration exhausted; marked for cleanup in `Idle()`
- **Pause/Resume**: `Pause` RAII guard calls `SetStatus(false)` ΓåÆ all players frozen; destructor ΓåÆ `SetStatus(true)` ΓåÆ resume

## Learning Notes

**Marathon/Aleph One Audio Philosophy**
- Sounds are identified by **integer index**, not filename; this allows MML mods to remap Sound_ButtonSuccess() to a different `.snd` file entry without code changes
- **Spatial audio without HRTF**: Uses simple angle-to-stereo-pan + distance-to-volume attenuation; no head-related transfer functions (HRTF), no Doppler, no reverb simulationΓÇöreflects 1990s real-time constraints
- **Fixed obstruction flag** rather than raycasting: `GetSoundObstructionFlags()` is a fast boolean check (likely "is source in same polygon as listener?"), not a gradual muffling curve; suggests deterministic gameplay priority over audio realism
- **Pre-allocated ambient channel limits**: Hardcoded `MAXIMUM_AMBIENT_SOUND_CHANNELS = 4` suggests design for mid-1990s audio hardware; modern engines would use voice stealing or unlimited channels

**Modern Engine Differences**
- Today's engines (Unreal, Unity, FMOD) support **streaming** rather than pre-cached decompression; SoundManager pre-loads entire sounds into RAM
- **No event system**: Direct function calls vs. event queuing; modern engines use async sound playback via event callbacks to avoid frame-rate coupling
- **No distance model configurability**: Attenuation is hardcoded; FMOD/Wwise expose curves for distance, air absorption, environmental effects
- **No voice prioritization**: No detailed voice stealing logic shown; modern engines rank voices by distance, category, age to decide which to drop when channel limit hit

## Potential Issues

1. **Race Condition Risk in Pause**  
   The `Pause` RAII class calls `SetStatus(true/false)` without apparent mutex protection. If a background SoundPlayer thread reads `active` flag while the main thread is in `SetStatus()`, a sound might start/stop mid-pause. The code likely relies on single-threaded access or implicit synchronization via SoundPlayer design.

2. **Sound Identifier Collision**  
   Multiple calls to `PlaySound(..., identifier)` with the same identifier may overwrite in `sound_players` set. If two entities both use identifier `123`, stopping one stops both. Unclear if identifiers are entity-unique or globally managed.

3. **Listener Position Sync Lag**  
   `UpdateListener()` is called once per frame from `Idle()`. If game updates run at 30 FPS and audio at 44.1 kHz, there's up to 33 ms of listener position stale data before stereo panning is recalculated. Fast-moving listeners in first-person view may perceive panning lag.

4. **Obstruction All-or-Nothing**  
   `GetSoundObstructionFlags()` returns a single flag, not a gradual attenuation curve. A sound either passes through or doesn't; no soft muffling based on wall thickness or distance to obstruction.
