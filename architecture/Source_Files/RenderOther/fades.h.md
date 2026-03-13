# Source_Files/RenderOther/fades.h

## File Purpose
Header file defining the interface for screen fade and color tinting effects in the Marathon/Aleph One game engine. Manages fade-in/fade-out transitions, damage tints (red for bullets, yellow for explosions, etc.), environmental tints (underwater, lava, sewage), and gamma correction. Supports XML-based configuration via MML parser.

## Core Responsibilities
- Define fade types for cinematics, damage feedback, and environmental effects
- Provide API to start, update, and stop fades
- Implement gamma correction for color tables
- Manage fade effect application with configurable delay (MacOS workaround)
- Parse and reset XML-based fade configuration
- Check fade completion and screen state

## Key Types / Data Structures
None (declarations only; external structs/classes referenced: `color_table`, `InfoTree`).

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_fades
- **Signature:** `void initialize_fades(void)`
- **Purpose:** Initialize fade system at engine startup
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets up fade state machine
- **Notes:** Called once during engine init

### update_fades
- **Signature:** `bool update_fades(bool game_in_progress = false)`
- **Purpose:** Advance fade state each frame
- **Inputs:** `game_in_progress` ΓÇö whether game is actively running
- **Outputs/Return:** Boolean (likely true if fade is still active)
- **Side effects:** Updates color tables, modifies screen rendering
- **Calls:** Likely modifies internal fade state

### start_fade / explicit_start_fade
- **Signature:** `void start_fade(short type)` / `void explicit_start_fade(short type, struct color_table *original_color_table, struct color_table *animated_color_table, bool game_in_progress = false)`
- **Purpose:** Begin a fade effect of specified type
- **Inputs:** Fade type enum; explicit version takes color table pointers
- **Outputs/Return:** None
- **Side effects:** Initializes fade animation state, modifies color tables
- **Notes:** Explicit version allows caller to provide color tables; implicit version uses internal tables

### set_fade_effect
- **Signature:** `void set_fade_effect(short type)`
- **Purpose:** Apply a fade/tint effect immediately (e.g., environmental tint)
- **Inputs:** Effect type (water, lava, sewage, JjaroGoo, goo)
- **Outputs/Return:** None
- **Side effects:** Modifies active color table

### gamma_correct_color_table
- **Signature:** `void gamma_correct_color_table(struct color_table *uncorrected_color_table, struct color_table *corrected_color_table, short gamma_level)`
- **Purpose:** Apply gamma correction to a color table
- **Inputs:** Source table, gamma level (0ΓÇô7)
- **Outputs/Return:** Corrected table populated
- **Side effects:** Modifies corrected_color_table

### parse_mml_faders / reset_mml_faders
- **Signature:** `void parse_mml_faders(const InfoTree& root)` / `void reset_mml_faders()`
- **Purpose:** Load/reset fade configuration from XML
- **Inputs:** InfoTree root node (XML parse tree)
- **Outputs/Return:** None
- **Side effects:** Updates internal fade definitions and callbacks
- **Notes:** Decouples XML parsing from C callbacks; enables OpenGL implementations

**Helper functions** (see notes): `stop_fade()`, `fade_finished()`, `get_actual_gamma_adjust()`, `fade_blacked_screen()`, `SetFadeEffectDelay()`.

## Control Flow Notes
- **Init:** `initialize_fades()` at startup; `parse_mml_faders()` during resource load
- **Frame:** `update_fades()` called each render frame to progress fade animation
- **Gameplay:** `start_fade()` / `set_fade_effect()` triggered by damage, environmental changes, or cinematic events; `SetFadeEffectDelay()` provides MacOS-specific workaround for dialog box artifacts
- **Shutdown:** Implied cleanup (no explicit shutdown function visible)

## External Dependencies
- **`struct color_table`** ΓÇö defined elsewhere; represents indexed color palette
- **`class InfoTree`** ΓÇö C++ class (likely XML parse tree); defined elsewhere
- **MML/XML infrastructure** ΓÇö supports fade configuration declaratively rather than hard-coded
