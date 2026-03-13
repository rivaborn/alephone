# Source_Files/RenderOther/fades.cpp

## File Purpose

Manages screen fade effects and color table manipulation for the Marathon/Aleph One game engine. Implements time-based visual transitions with various blend modes (tint, dodge, burn, negate, etc.), supports environmental tinting (water/lava/goo), handles gamma correction, and provides both software and OpenGL rendering backends.

## Core Responsibilities

- Initialize, update, and manage fade state with time-based interpolation
- Apply six distinct color manipulation effects with configurable opacity and duration
- Implement priority-based fade scheduling (higher priority fades interrupt lower ones)
- Support independent environmental effects (water, lava, sewage, Jjaro, alien goo) overlaid with fades
- Perform gamma correction on color tables for display calibration
- Integrate OpenGL faders via queue entries for hardware-accelerated effects
- Parse and reset MML-customizable fade definitions and environmental effects

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `fade_definition` | struct | Defines a fade type: function ptr, color, initial/final transparency, duration, flags, priority |
| `fade_effect_definition` | struct | Environmental effect mapping: which fade to apply and its opacity |
| `fade_data` | struct | Runtime state: active flag, fade/effect type, last update ticks, source/dest color tables |
| `OGL_Fader` | struct (imported) | OpenGL fader queue entry: type, RGBA color channels |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `fade` | `fade_data*` | static | Current active fade state (NULL or pointer to singleton) |
| `last_fade_type` | `short` | static | Tracks most recent fade for `fade_blacked_screen()` query |
| `fades_random_seed` | `uint16` | static | PRNG state for randomize effect (via FADES_RANDOM macro) |
| `CurrentOGLFader` | `OGL_Fader*` | static | Points to active OpenGL fader queue entry; NULL if OpenGL inactive |
| `FadeEffectDelay` | `int` | static | Counter to delay effect updates (MacOS workaround) |
| `fade_definitions` | `fade_definition[NUMBER_OF_FADE_TYPES]` | static | Array of ~32 fade type definitions (customizable via MML) |
| `fade_effect_definitions` | `fade_effect_definition[NUMBER_OF_FADE_EFFECT_TYPES]` | static | Environmental effect mappings (customizable via MML) |
| `actual_gamma_values` | `float[NUMBER_OF_GAMMA_LEVELS]` | static | Lookup table (0.70ΓÇô1.30) for gamma correction |
| `original_fade_definitions` | `fade_definition*` | static | Backup of defaults for MML reset; allocated on first parse |
| `original_fade_effect_definitions` | `fade_effect_definition*` | static | Backup of defaults for MML reset; allocated on first parse |

## Key Functions / Methods

### initialize_fades
- **Signature:** `void initialize_fades(void)`
- **Purpose:** Allocate and zero-initialize the global `fade` state structure.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates `fade` heap object; clears flags and sets effect type to NONE.
- **Calls:** Standard allocator (`new`)
- **Notes:** Must be called once at startup. No-op if called multiple times.

### update_fades
- **Signature:** `bool update_fades(bool game_in_progress = false)`
- **Purpose:** Advance active fade by one frame, interpolate transparency, and update screen color table.
- **Inputs:** `game_in_progress` ΓÇô if true, use game ticks; else use machine ticks.
- **Outputs/Return:** `true` if fade is still active after update; `false` if complete.
- **Side effects:** Updates `fade->last_update_tick` and `last_update_game_tick`; calls `recalculate_and_display_color_table()`; records frame to Movie system.
- **Calls:** `get_fade_definition()`, `recalculate_and_display_color_table()`, `Movie::instance()->AddFrame()`
- **Notes:** Interpolates `transparency` linearly from initial to final over `period`. Handles `_random_transparency_flag` by adding random noise. Deactivates fade when `phase >= period`.

### start_fade
- **Signature:** `void start_fade(short type)`
- **Purpose:** Begin a fade effect using the world color table (convenience wrapper).
- **Inputs:** `type` ΓÇô fade type index
- **Outputs/Return:** None
- **Side effects:** Calls `explicit_start_fade()` with `world_color_table` and `visible_color_table`.
- **Calls:** `explicit_start_fade()`
- **Notes:** Simple wrapper for typical in-game usage.

### explicit_start_fade
- **Signature:** `void explicit_start_fade(short type, struct color_table *original_color_table, struct color_table *animated_color_table, bool game_in_progress = false)`
- **Purpose:** Core fade initiation with priority checking and restart throttling.
- **Inputs:** `type` ΓÇô fade index; color tables for source/dest; `game_in_progress` ΓÇô timing mode.
- **Outputs/Return:** None
- **Side effects:** Sets `fade->type`, `last_update_tick/game_tick`, color table pointers; activates fade if conditions pass.
- **Calls:** `get_fade_definition()`, `recalculate_and_display_color_table()`
- **Notes:** Implements priority logic: a new fade only starts if (1) no fade active, (2) its priority ΓëÑ current fade's, or (3) it's the same type but throttle time (`MINIMUM_FADE_RESTART`) has elapsed. Prevents rapid re-triggering. Does not activate if `period == 0` (instant fade).

### stop_fade
- **Signature:** `void stop_fade(void)`
- **Purpose:** Immediately end the active fade and apply its final color state.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears fade active flag; applies final transparency and redraws color table.
- **Calls:** `get_fade_definition()`, `recalculate_and_display_color_table()`
- **Notes:** Safe to call if no fade is active (early return).

### fade_finished
- **Signature:** `bool fade_finished(void)`
- **Purpose:** Query whether the current fade is complete.
- **Inputs:** None
- **Outputs/Return:** `true` if no fade active; `false` if fade in progress.
- **Side effects:** None
- **Calls:** None
- **Notes:** Inverse of `FADE_IS_ACTIVE(fade)`.

### set_fade_effect
- **Signature:** `void set_fade_effect(short type)`
- **Purpose:** Set or clear an environmental tint effect (e.g., underwater, lava).
- **Inputs:** `type` ΓÇô effect index or NONE to clear.
- **Outputs/Return:** None
- **Side effects:** Updates `fade->fade_effect_type`; may invoke `recalculate_and_display_color_table()` or `animate_screen_clut()`.
- **Calls:** `get_fade_effect_definition()`, `recalculate_and_display_color_table()`, `animate_screen_clut()`, `SetOGLFader()`
- **Notes:** Respects `FadeEffectDelay` counter (decrements if > 0, skips update). If no active fade, applies effect immediately to world color table and broadcasts to OpenGL.

### full_fade
- **Signature:** `void full_fade(short type, struct color_table *original_color_table)`
- **Purpose:** Synchronous fade: starts a fade and blocks until completion (for cinematics).
- **Inputs:** `type` ΓÇô fade index; source color table.
- **Outputs/Return:** None
- **Side effects:** Modifies `animated_color_table` (local copy); calls `update_fades()` in a loop.
- **Calls:** `obj_copy()`, `explicit_start_fade()`, `update_fades()`, `Music::instance()->Idle()`
- **Notes:** Used for intro/outro sequences. Spins `update_fades()` until fade completes, yielding to music idle.

### gamma_correct_color_table
- **Signature:** `void gamma_correct_color_table(struct color_table *uncorrected_color_table, struct color_table *corrected_color_table, short gamma_level)`
- **Purpose:** Apply gamma correction curve to a color table (for display calibration).
- **Inputs:** Source color table, gamma level (0ΓÇô7).
- **Outputs/Return:** Fills `corrected_color_table`.
- **Side effects:** None (pure computation).
- **Calls:** `pow()` (math library)
- **Notes:** Raises each RGB component to power `gamma_value`. If gamma Γëê 1.0, memcpys unchanged. If recording movie, forces gamma = 1.0.

### get_actual_gamma_adjust
- **Signature:** `float get_actual_gamma_adjust(short gamma_level)`
- **Purpose:** Query the gamma multiplier for a given level.
- **Inputs:** `gamma_level` (0ΓÇô7)
- **Outputs/Return:** Gamma value (typically 0.70ΓÇô1.30).
- **Side effects:** None
- **Calls:** None
- **Notes:** Simple lookup; validates array bounds implicitly.

### fade_blacked_screen
- **Signature:** `bool fade_blacked_screen(void)`
- **Purpose:** Check if screen is currently black due to cinematic fade.
- **Inputs:** None
- **Outputs/Return:** `true` if last fade was `_start_cinematic_fade_in` or `_cinematic_fade_out` and no fade is now active.
- **Side effects:** None
- **Calls:** None
- **Notes:** Used to suppress HUD or menu rendering when screen is black.

### tint_color_table, randomize_color_table, negate_color_table, dodge_color_table, burn_color_table, soft_tint_color_table
- **Signature:** `static void [function_name](struct color_table *original, struct color_table *animated, struct rgb_color *color, _fixed transparency)`
- **Purpose:** Apply specific color blend modes to each color in the table.
- **Inputs:** Original color table, tint/blend color, opacity (0ΓÇôFIXED_ONE).
- **Outputs/Return:** Fills `animated` table.
- **Side effects:** If `CurrentOGLFader` is set, delegates to OpenGL and returns early; else modifies `animated` in-place.
- **Calls:** (OpenGL path) `TranslateToOGLFader()`; (software path) bit shifts and arithmetic
- **Notes:** 
  - **tint** ΓÇô linear interpolation between original and color.
  - **randomize** ΓÇô adds random noise masked by transparency.
  - **negate** ΓÇô XOR-based inversion with clamping.
  - **dodge** ΓÇô inverse multiplicative blend (screen dodge).
  - **burn** ΓÇô multiplicative blend (screen burn).
  - **soft_tint** ΓÇô tint based on original intensity.

### SetOGLFader
- **Signature:** `void SetOGLFader(int Index)`
- **Purpose:** Activate or deactivate an OpenGL fader queue entry.
- **Inputs:** Queue index (FaderQueue_Liquid or FaderQueue_Other).
- **Outputs/Return:** None
- **Side effects:** Sets `CurrentOGLFader` to queue entry or NULL depending on OpenGL active status.
- **Calls:** `OGL_FaderActive()`, `GetOGL_FaderQueueEntry()`
- **Notes:** Called before applying each effect layer. Clears fader type to NONE.

### TranslateToOGLFader
- **Signature:** `static void TranslateToOGLFader(rgb_color &Color, _fixed Opacity)`
- **Purpose:** Convert fixed-point RGB and opacity to float for OpenGL fader.
- **Inputs:** RGB color (16-bit per channel), opacity (fixed-point).
- **Outputs/Return:** Fills `CurrentOGLFader->Color[0ΓÇô3]` (RGBA as float).
- **Side effects:** Updates `CurrentOGLFader` array.
- **Calls:** None
- **Notes:** Assumes `CurrentOGLFader` is non-NULL.

### recalculate_and_display_color_table
- **Signature:** `static void recalculate_and_display_color_table(short type, _fixed transparency, struct color_table *original, struct color_table *animated, bool fade_active)`
- **Purpose:** Apply both environmental effect and fade effect to color table, then send to screen/OpenGL.
- **Inputs:** Fade type, transparency, color tables, fade active flag.
- **Outputs/Return:** None
- **Side effects:** Calls effect/fade procs; modifies `animated`; calls `animate_screen_clut()` or OpenGL renderer.
- **Calls:** `SetOGLFader()`, `get_fade_definition()`, `animate_screen_clut()`, `draw_intro_screen()`, color procs.
- **Notes:** Applies effect first, then fade. Chains color tables (effect output feeds fade input). Flushes to screen and menu if in main menu.

### parse_mml_faders
- **Signature:** `void parse_mml_faders(const InfoTree& root)`
- **Purpose:** Parse and apply custom fade definitions from MML/XML.
- **Inputs:** InfoTree root with `<fader>` and `<liquid>` children.
- **Outputs/Return:** None
- **Side effects:** Modifies `fade_definitions` and `fade_effect_definitions` arrays; allocates and fills backup arrays on first call.
- **Calls:** `InfoTree::read_*()` methods, effect/fade function pointer assignment.
- **Notes:** Supports customizing fade proc, color, opacity, period, flags, priority. Backups defaults for `reset_mml_faders()`.

### reset_mml_faders
- **Signature:** `void reset_mml_faders()`
- **Purpose:** Restore fade definitions to defaults, clearing MML customizations.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Copies backed-up definitions back to main arrays; frees backups.
- **Calls:** Standard allocator (`free`)
- **Notes:** Safe to call if no MML was ever parsed (backups are NULL).

## Control Flow Notes

- **Initialization**: `initialize_fades()` called once at engine startup.
- **Main loop**: `update_fades()` called each frame; checks active flag and interpolates fade progress.
- **User input/game events**: Code calls `start_fade()` or `explicit_start_fade()` to trigger fades; `set_fade_effect()` to set environmental tints.
- **Priority arbitration**: `explicit_start_fade()` only activates if new fade outranks current or passes restart throttle.
- **Dual-layer effects**: Environmental effect (e.g., water tint) is applied first, then the main fade effectΓÇöboth layers use the same color-manipulation function signature.
- **OpenGL integration**: Each layer checks `CurrentOGLFader` before software rendering; if set, delegates blend mode and color to GPU.
- **Shutdown**: No explicit cleanup; fade struct exists for engine lifetime.

## External Dependencies

- **cseries.h** ΓÇô core macros (PIN, MAX, GetMemberWithBounds), constants (FIXED_ONE, FIXED_FRACTIONAL_BITS).
- **fades.h** ΓÇô public API and enum definitions (fade types, effect types, fader function types).
- **screen.h** ΓÇô color table management (`animate_screen_clut`, `draw_intro_screen`, `world_color_table`, `visible_color_table`, `interface_color_table`).
- **interface.h** ΓÇô game state queries (`get_game_state`).
- **map.h** ΓÇô timing constant (`TICKS_PER_SECOND`, `MACHINE_TICKS_PER_SECOND`); dynamic world state (`dynamic_world->tick_count`).
- **InfoTree.h** ΓÇô XML parser for MML support.
- **OGL_Faders.h** ΓÇô OpenGL fader integration (`OGL_FaderActive`, `GetOGL_FaderQueueEntry`, fader queue indices).
- **Music.h** ΓÇô music system for `full_fade()` idle loop.
- **Movie.h** ΓÇô movie recording integration (`Movie::instance()->AddFrame`).
- **Standard library** ΓÇô `<string.h>`, `<stdlib.h>`, `<math.h>` (pow, memcpy); `<limits.h>` (SHRT_MAX).
