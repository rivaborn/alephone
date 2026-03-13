# Source_Files/Sound/SoundManager.cpp - Enhanced Analysis

## Architectural Role

SoundManager is the **primary integration hub** between the game world simulation (GameWorld) and the audio backend (OpenAL). It sits at a crucial crossroads: translating dynamic world events (entity movement, firing weapons, ambient environment) into 3D spatialised audio parameters, managing sound data lifecycle under memory constraints, and coordinating frame-by-frame updates to keep audio synchronized with 30 FPS game ticks. This file embodies the design principle that audio is a **first-class simulation subsystem**, not an afterthoughtΓÇöit receives callbacks from world geometry queries, tracks source locations every frame, and even coordinates with Movie recording for deterministic audio timestamps.

## Key Cross-References

### Incoming (who depends on this)

**Direct callers:**
- **GameWorld/marathon2.cpp** ΓÇô Main game loop calls `Idle()` each frame; registers callbacks (`_sound_listener_proc`, `_sound_obstructed_proc`, `_sound_add_ambient_sources_proc`) for world-state queries
- **GameWorld/map.cpp** ΓÇô Provides `_sound_obstructed_proc()` callback; queries polygon/line obstruction for sound filtering
- **RenderMain/render.cpp** ΓÇô May trigger visual-audio sync (explosions, effect playback)
- **Network/network_games.cpp** ΓÇô Multiplayer code calls `PlaySound()` for networked events
- **Lua scripting** ΓÇô Game scripts invoke `PlaySound()`, `StopSound()`, volume control
- **Misc/interface.cpp, shell.cpp** ΓÇô UI sounds via `DirectPlaySound()` (beeps, alerts)
- **Misc/preferences_widgets_sdl.cpp** ΓÇô Volume adjustment (`AdjustVolumeUp/Down`)

**Global state readers:**
- `parameters` (sound configuration) read by shell options, Lua, preferences
- `sound_file`, `sound_source` read for diagnostic/debug output

### Outgoing (what this file calls)

**Primary subsystem dependencies:**
- **OpenALManager** (singleton) ΓÇô `Get()`, `SetMasterVolume()`, `UpdateListener()`, `GetElapsedPauseTime()`, `IsPaused()`; manages OpenAL context, source pool, 3D listener state
- **SoundPlayer** (per-voice state machine) ΓÇô created via `ManageSound()`, updated in `ManagePlayers()`, stopped in `StopSound()`, `StopAllSounds()`
- **SoundFile hierarchy** (M1SoundFile, M2SoundFile) ΓÇô `Open()`, `SourceCount()`, `GetSoundData()`, `Close()`
- **SoundMemoryManager** (private LRU pool) ΓÇô `Add()`, `Get()`, `IsLoaded()`, `Release()`, `Update()`; auto-evicts oldest sounds when budget exceeded
- **SoundDefinition** (from sound_definitions.h) ΓÇô metadata queries via `GetSoundDefinition()`, flags (`_sound_is_ambient`, `_sound_cannot_be_obstructed`)
- **SoundReplacements** (singleton) ΓÇô `GetSoundOptions()` for per-permutation external audio overrides
- **ReplacementSounds/SoundsPatch** ΓÇô `get_sound_data()` for XML-driven sound patches
- **InfoTree (XML)** ΓÇô `parse_mml_sounds()` parses sound patch definitions
- **Movie** (recording) ΓÇô `IsRecording()`, `GetCurrentAudioTimeStamp()` for deterministic playback
- **World model** ΓÇô `world_location3d` (source/listener positions), `_sound_listener_proc()` callback
- **FileSpecifier** ΓÇô path resolution, file existence checks

## Design Patterns & Rationale

| Pattern | Implementation | Rationale |
|---------|---|---|
| **Singleton** | `SoundManager::instance()`, `OpenALManager::Singleton` | Single audio subsystem manages global device state; prevents multiple initialization/cleanup cycles |
| **LRU Memory Manager** | `SoundMemoryManager` with `last_played` timestamps; `ReleaseOldestSound()` | Constrained embedded memory; predictable eviction without stalling audio thread |
| **Lazy Loading** | `LoadSound()` defers load until first `PlaySound()` call | Reduces startup time; distributes I/O cost across gameplay |
| **Strategy Pattern** | Multiple `PlaySound()` overloads (short index, LoadedResource, direct) | Supports UI sounds, game world sounds, streaming sources without code duplication |
| **Observer Callback** | `SoundReleased` callback in `SoundMemoryManager::Release()` | Decouples memory eviction notification from sound playback logic |
| **Weak Pointer Pool** | `sound_players` vector with `std::shared_ptr<SoundPlayer>` | Automatic cleanup via `CleanInactivePlayers()` when sounds finish; avoids dangling pointers |
| **Parameter Object** | `SoundParameters` struct (pitch, stereo, obstruction, location) | Avoids function signature bloat; groups related data; facilitates updates in `ManagePlayers()` |
| **Facade** | `SoundManager` hides OpenAL, SoundPlayer, SoundFile complexity | Game code calls `PlaySound(sound_index, location)` without knowing OpenAL source lifecycle |

**Rationale for complex GetSoundPlayer() logic:**
- **Multi-identifier tracking**: Supports both "sound index" (what is playing) and "source identifier" (where, e.g., monster #5) lookups
- **Priority-based eviction**: When MAX_SOUNDS_FOR_SOURCE hit, culls weakest priority sound, not just first-come-first-served
- **sound_identifier_only flag**: UI sounds don't care about source; monsters do

## Data Flow Through This File

### Cold Path: Load Phase
```
Initialize(parameters)
  Γåô OpenSoundFile(FileSpecifier)
    Γö£ΓöÇ Try M2SoundFile::Open()
    ΓööΓöÇ Fall back M1SoundFile::Open()
  Γåô SetParameters() ΓåÆ SetStatus(true) ΓåÆ OpenALManager activated
```

### Hot Path: Per-Frame Update
```
GameWorld/marathon2.cpp: update_world()
  Γö£ΓöÇ [Game logic: monsters move, player fires, platforms activate]
  Γö£ΓöÇ SoundManager::Idle()
  Γöé   Γö£ΓöÇ UpdateListener() ΓåÆ listener position to OpenAL
  Γöé   Γö£ΓöÇ CauseAmbientSoundSourceUpdate()
  Γöé   Γöé   ΓööΓöÇ UpdateAmbientSoundSources() ΓåÆ accumulate active ambient sources, cull by priority
  Γöé   ΓööΓöÇ ManagePlayers()
  Γöé       ΓööΓöÇ For each SoundPlayer:
  Γöé           Γö£ΓöÇ If dynamic_source_location3d set, update position
  Γöé           Γö£ΓöÇ Query GetSoundObstructionFlags() from map geometry
  Γöé           ΓööΓöÇ If changed, SoundPlayer::UpdateParameters()
  ΓööΓöÇ [Audio thread: OpenAL updates source positions, volume, filtering]
```

### Hot Path: Oneshot Sound
```
GameWorld code calls: PlaySound(sound_index, source_location, identifier, pitch, soft_rewind)
  Γö£ΓöÇ LoadSound(sound_index)
  Γöé   Γö£ΓöÇ Check sounds->IsLoaded()
  Γöé   Γö£ΓöÇ Try sounds_patches.get_sound_data() (XML patches)
  Γöé   Γö£ΓöÇ Fall back sound_file->GetSoundData() (built-in)
  Γöé   Γö£ΓöÇ Query SoundReplacements::GetSoundOptions() (external audio files)
  Γöé   ΓööΓöÇ sounds->Add(data, index, slot) ΓåÆ may trigger ReleaseOldestSound() if budget exceeded
  Γö£ΓöÇ CalculateInitialSoundVariables() ΓåÆ volume, stereo panning from distance/obstruction
  Γö£ΓöÇ GetSoundObstructionFlags() ΓåÆ query wall/media state
  Γö£ΓöÇ BufferSound(parameters) ΓåÆ load permutation, create SoundHeader
  ΓööΓöÇ ManageSound() ΓåÆ reuse SoundPlayer or create new via OpenALManager
```

### Hot Path: Ambient Sounds
```
UpdateAmbientSoundSources() [called from Idle() if ambient flag set]
  Γö£ΓöÇ Invoke _sound_add_ambient_sources_proc() callback
  Γö£ΓöÇ Iterate returned ambient_sound_data[] entries
  Γö£ΓöÇ For each active source:
  Γöé   Γö£ΓöÇ LoadSound() if not cached
  Γöé   Γö£ΓöÇ BufferSound() ΓåÆ create or reuse SoundPlayer
  Γöé   ΓööΓöÇ Add to ambient_sound_players set
  Γö£ΓöÇ Cull weakest sources if count > MAXIMUM_AMBIENT_SOUND_CHANNELS
  ΓööΓöÇ UpdateParameters() on remaining players
```

### Data Persistence: Unload Phase
```
UnloadSound(sound_index)
  Γö£ΓöÇ StopSound() ΓåÆ find matching SoundPlayer, AskStop()
  ΓööΓöÇ sounds->Release() ΓåÆ decrement memory, invoke SoundReleased callback

SoundMemoryManager::ReleaseOldestSound() [triggered on Add if memory exceeded]
  Γö£ΓöÇ Scan m_entries for oldest last_played timestamp
  Γö£ΓöÇ Release() oldest ΓåÆ invoke SoundReleased callback
  ΓööΓöÇ Retry Add()
```

## Learning Notes

### What a Developer Studies From This File

1. **Integration architecture**: How to coordinate subsystems (game world ΓåÆ audio backend) via callbacks and per-frame updates; the importance of **deterministic frame ticking** for audio sync

2. **3D audio spatialization**: The gap between "3D sounds enabled" (OpenAL HRTF) vs. "3D sounds disabled" (stereo panning via angle/distance); understanding when to use obstruction flags vs. volume attenuation

3. **Memory budget discipline**: LRU eviction under constraint; the trade-off between keeping frequently-played sounds vs. freeing space for new ones; why `last_played` timestamp matters more than load frequency

4. **Compatibility layers**: Supporting Marathon 1 M1 formats alongside Marathon 2 M2 formats; graceful fallback (`try M2, then M1`); MML patching system for modding without engine recompilation

5. **Parameter calculation**: Converting world state (distance, angle, obstruction) into audio parameters (volume, panning, filtering); understanding non-linear volume curves (`distance_to_volume`)

6. **Idempotent design**: `LoadSound()` is safe to call multiple times; `Update()` just refreshes `last_played` without reloading; prevents resource leaks from repeated calls

### Idiomatic Patterns This Engine Uses

- **Callback architecture**: Game world registers `_sound_listener_proc()`, `_sound_obstructed_proc()`, `_sound_add_ambient_sources_proc()` callbacks instead of exposing global world state
- **Parameter structs instead of getter/setter chains**: `SoundParameters`, `SoundVolumes` package related data
- **Per-frame update loops**: Explicit `Idle()` call from main loop, not implicit background threads (easier to reason about timing)
- **Enum-based type tags**: `sound_code`, `sound_source` stored as shorts, not type-safe enums (legacy Marathon 1 compatibility)
- **Magic number constants**: `MAXIMUM_AMBIENT_SOUND_CHANNELS`, `MAXIMUM_SOUND_VOLUME` defined in headers, not configurable

### Modern Engines Do Differently

- **Real-time audio graphs** (FMOD, Wwise): Declarative audio mixing trees instead of imperative `PlaySound()` calls
- **Async resource loading**: Sound loading on separate thread with future/promise, not blocking main thread
- **ECS-based source management**: Audio sources as entities with components, not pooled arrays
- **Shader-based spatialization**: GPU-computed HRTF instead of OpenAL callbacks
- **Unified time base**: Synchronized audio/video via shared high-resolution tick counter, not separate movie logic

## Potential Issues

| Issue | Severity | Context |
|-------|----------|---------|
| **GetSoundPlayer() complexity** | Medium | Nested conditionals in priority selection could miss edge cases (e.g., tie-breaking when multiple sounds have same priority) |
| **SoundMemoryManager eviction timing** | Medium | ReleaseOldestSound() happens during Add(), blocking the calling thread; consider async eviction or larger buffer to amortize |
| **Dynamic source tracking dangling pointer** | High | `parameters.dynamic_source_location3d` assumes world_location3d pointer remains valid frame-to-frame; if source dies mid-play, undefined behavior |
| **No validation of SoundDefinition flags** | Low | LoadSound() trusts GetSoundDefinition(); if definition missing or corrupt, returns false silently; error could be detected earlier |
| **Ambient sound cull order** | Low | Cull by lowest priority first; ties broken arbitrarily by `std::min_element`; could prioritize by distance/proximity instead |
| **MML patch global state mutation** | Medium | parse_mml_sounds() modifies global ambient/random/dialog_sound_definitions; no mechanism to prevent overlapping patches or rollback partial failures |
| **Movie recording timestamp sync** | Low | GetCurrentAudioTick() special-cases recording; if movie timestamp lags, audio might desync from visual |
| **No per-sound rate limiting** | Low | Multiple rapid calls to PlaySound() same index may create redundant SoundPlayer instances; could implement re-trigger suppression |
| **StopSound(NONE, index) ambiguity** | Low | identifier=NONE matches any playing instance; could stop wrong sound if intent was "stop all instances of this sound" |

---

**Integration Insights:**
- SoundManager is **tightly coupled to GameWorld update loop** via `Idle()` callback; this is intentional for frame-synchronization but makes audio hard to decouple for testing or headless scenarios
- **Memory pressure from concurrent ambient + oneshot sounds** could cause thrashing if ambient sources stay loaded; consider prioritizing oneshot sounds or increasing budget for dynamic contexts
- **3D audio fidelity depends on callback quality**: If `_sound_obstructed_proc()` is slow, manifests as audio lag; profiling is critical
- **XML patching system is powerful but fragile**: Malformed MML silently falls back to built-ins; would benefit from validation warnings
