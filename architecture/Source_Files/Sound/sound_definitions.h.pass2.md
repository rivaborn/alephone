# Source_Files/Sound/sound_definitions.h - Enhanced Analysis

## Architectural Role

This file implements the **sound configuration layer** bridging the resource manager (Files subsystem) and the runtime audio engine (SoundManager). It provides three independent lookup tables (`sound_behavior_definitions`, `ambient_sound_definitions`, `random_sound_definitions`) that partition the sound system into behavior-driven categories. The attenuation curves defined here directly control 3D spatial audio positioningΓÇödistance-to-volume mappings that are evaluated frame-by-frame by SoundManager during playback. The file also defines the persistent binary format (`sound_file_header`, `sound_definition`) that maps disk-resident sound data to runtime structures.

## Key Cross-References

### Incoming (files that depend on this)

- **`SoundManager.h/cpp`**: The primary consumer. Uses `sound_behavior_definitions` for attenuation lookup, reads `permutations` and `chance` fields to drive playback decisions, consults `last_played` for restart throttling, and dereferences the `ptr` + `sound_offsets` array to fetch audio samples.
- **`SoundsPatch.h`**: Patch/addon system that rebuilds sound definitions at runtime; adds/overrides entries.
- **`game_wad.cpp`**: Saves/restores `sound_definition` structures during game persistence (save/load game).
- **`Sound/SoundPlayer.h/cpp`**: Consumes the binary format via `sound_file_header` during resource file parsing.

### Outgoing (what this file depends on)

- **`SoundManagerEnums.h`**: Supplies `NUMBER_OF_AMBIENT_SOUND_DEFINITIONS`, `NUMBER_OF_RANDOM_SOUND_DEFINITIONS`, `NUMBER_OF_SOUND_DEFINITIONS` (array bounds), and all `_snd_*`, `_ambient_snd_*`, `_random_snd_*` enum codes.
- **`world.h`**: Provides `WORLD_ONE` constant for attenuation distance thresholds (e.g., `5*WORLD_ONE` Γëê 5 game units for quiet sound falloff).
- **Implicit global**: `MAXIMUM_SOUND_VOLUME` (from SoundManagerEnums), `FIXED_ONE` (fixed-point unit from CSeries).

## Design Patterns & Rationale

**Tripartite Sound System**: Separates ambient (looping, conditional), random (sporadic events), and event-driven (weapons, actions) sounds into distinct lookup paths. This allows SoundManager to handle each category with different scheduling strategiesΓÇöambient sounds loop based on location presence, random sounds fire asynchronously, event sounds play synchronously from game logic.

**Distance Attenuation via Lookup Curves**: Rather than computing attenuation per-frame, `depth_curve_definition` precomputes a two-point linear interpolation: `(max_volume at distance_0, min_volume at distance_N)`. This avoids expensive per-frame math, favoring 1990s CPU budgets. Two curves per behavior (obstructed vs. unobstructed) let footstep-adjacent walls attenuate differently than open-air sounds.

**Probability Gating**: Sound trigger logic uses `uint16 chance` as a fixed-point threshold (`AbsRandom() >= chance` determines playback). This is counterintuitive to modern code (higher `chance` means *less* likely), reflecting legacy design choices. The pre-computed constants (`_fifty_percent = 32768*5/10`) encode probability as quantized 16-bit fixed-point.

**Permutation System**: Stores up to 5 sound variants per definition with individual file offsets (`sound_offsets[MAXIMUM_PERMUTATIONS_PER_SOUND]`), enabling cheap variety without explosion of sound codes. `permutations_played` tracks round-robin cycling or random selection state.

## Data Flow Through This File

1. **Load**: `SoundPlayer` reads `sound_file_header` from disk; iterates `source_count ├ù sound_count` `sound_definition` entries; populates `ptr` (allocated audio sample buffer) and `sound_offsets` (within-buffer permutation positions).

2. **Runtime Query**: Each frame, SoundManager looks up behavior via `sound_definitions[sound_code].behavior_index` ΓåÆ `sound_behavior_definitions[behavior_index]` ΓåÆ obstructed/unobstructed curve ΓåÆ distance-to-volume mapping.

3. **Playback Decision**: Checks `chance` field; if random threshold passed and `(last_played + restart_threshold) < now`, initiates playback on an unused channel.

4. **Permutation Picking**: Selects a permutation index (round-robin or random), retrieves offset from `sound_offsets[]`, plays from `ptr + offset` for `single_length` samples.

## Learning Notes

- **Era Artifact**: The struct layout and field ordering (int16 codes, flags, fixed-point pitch) reflects Marathon's 1994 macOS origins and has been preserved for Infinity compatibility (see comment: `SOUND_FILE_TAG should not be changed from 'snd2'`).
- **Fixed-Point Pitch**: `low_pitch` / `high_pitch` use `_fixed` (fixed-point 16.16), encoding playback rates relative to `FIXED_ONE`. Zero means "use default" or "use low_pitch", allowing compact parameterization.
- **Permutation-Based Variety**: Modern engines randomize pitch/timing; this engine pre-records variants. Cheaper but less dynamic.
- **Obstructed Curve Design**: Quiet sounds (water, sewage) return `{0, 0, 0, 0}` for obstructed case, meaning walls silence them entirelyΓÇöa coarse but effective spatial cue.

## Potential Issues

- **Hardcoded Permutation Limit**: `MAXIMUM_PERMUTATIONS_PER_SOUND = 5` wastes array space (5 offsets per sound) yet silently caps sound variety. Increasing this ripples across binary file format and struct size.
- **Mixed Concerns in `sound_definition`**: Runtime mutable state (`last_played`, `permutations_played`) lives alongside immutable definition fields. Refactoring would separate definition (const) from playback state (mutable).
- **Probability Semantics**: The "backwards" logic (`chance >=` instead of `<=`) has likely caused off-by-one bugs in downstream code; `_always = 0` is non-obvious.
- **No Format Versioning in `SOUND_FILE_TAG`**: Hard-coded 'snd2' tag prevented sound format evolution; any future enhancement (e.g., compressed audio, more permutations) breaks compatibility.
