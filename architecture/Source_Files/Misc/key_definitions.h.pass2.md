# Source_Files/Misc/key_definitions.h - Enhanced Analysis

## Architectural Role

This file defines the bridge between SDL hardware input (scancodes) and the game's semantic action layer (player physics flags). As a data-driven configuration header, it serves as the canonical reference keyboard layout for the input subsystem, enabling multiple keyboard configurations (standard, left-handed, PowerBook) to coexist without code duplication. The file sits at the boundary between the **Input subsystem** (hardware abstraction, SDL layer) and **GameWorld** (action interpretation, physics simulation), making it crucial for deterministic frame-by-frame input processing in `vbl.c`.

## Key Cross-References

### Incoming (who depends on this file)
- **vbl.c** (mentioned in header comment as sole consumer): The vertical-blank/frame-update handler polls `standard_key_definitions` each frame during input polling, converting hardware state to player action flags
- **Input subsystem** (indirectly): Joystick/mouse modules map controller input to comparable scancode ranges for unified action mapping
- **Preferences system**: User key rebinding reads/writes modifications to arrays with the same structure and order as `standard_key_definitions`

### Outgoing (what this file depends on)
- **player.h** action flag macros: All 21 action constants (`_moving_forward`, `_left_trigger_state`, `_toggle_map`, etc.) are defined elsewhere and consumed as semantic actions
- **SDL2 scancode enum**: `SDL_Scancode` type provides hardware-independent scancode identifiers
- **interface.h, player.h**: Header includes for type definitions and macro exports

## Design Patterns & Rationale

**Data-driven configuration with compile-time array sizing:**
The `NUMBER_OF_STANDARD_KEY_DEFINITIONS` macro uses `sizeof(array) / sizeof(element)` arithmetic. This is robustΓÇöchanges to the array automatically update the count, and the compiler validates the array is non-empty. This pattern avoids sentinel values or manual counts prone to drift.

**Multiple keyboard layout arrays (inferred from comments):**
The header comment explicitly notes "various key setups that the user can get" and warns that "arrays must all be in the same order." This suggests `standard_key_definitions` is one of several layout arrays (left-handed, PowerBook, etc.) defined elsewhere, all with identical struct ordering for parallel indexing during UI dialogs and preference loading. This is a lightweight polymorphism pattern for pre-computed configurations.

**Semantic action abstraction:**
Rather than storing raw SDL keycodes, the file maps to player-facing action flags. This decouples input polling logic from semantic meaning, allowing physics and AI to reason about "moving forward" rather than "key #X pressed."

**Rationale for structure of unused types:**
`blacklist_data` and `special_flag_data` are defined but not instantiated in this file. Their presence suggests planned features (key combination blacklisting to prevent hardware ghosting, or multi-tap/hold flag behaviors) that may be realized in parallel lookup tables elsewhere or via Lua scripting.

## Data Flow Through This File

1. **Input polling** (each frame in vbl.c):
   - Hardware state (keyboard, joystick, mouse) ΓåÆ SDL layer produces scancode bitmask
   - Frame loop iterates through `standard_key_definitions` array
   - For each entry, check if scancode is active in current hardware state
   - Accumulate matching `action_flag` into per-frame action word
   
2. **Action flag propagation**:
   - Accumulated action flags ΓåÆ `player_t` structure
   - Passed to `process_action_flags()` (GameWorld physics layer)
   - Physics engine converts discrete flags into continuous motion/aiming vectors (player velocity, look direction)

3. **Initialization / customization**:
   - Game startup: Load selected keyboard layout (standard, left-handed, etc.) into input subsystem
   - User rebinding in preferences: Read current layout, modify specific scancode ΓåÆ flag mappings, persist
   - Order constraint ensures UI dialog edit boxes align with array element positions

## Learning Notes

**1990s FPS input design idioms:**
- Numeric keypad movement (KP_8/4/6/5) reflects ubiquity of mechanical keyboards with dedicated keypads; modern laptops abandoned this
- WASD-adjacent secondary layout (Z/X for strafe, A/S for look, D/C/V for vertical look) is a common ergonomic compromise when WASD is occupied by movement
- Multiple layout support (standard, left-handed, PowerBook) reflects pre-standardization era when input hardware was less uniform

**Microphone button (`_microphone_button`):**
Unique to Marathon's network play design (voice chat in a 1994 FPS). Modern engines bundle voice I/O at a higher level (OS integration, audio subsystems), while Marathon exposed it as a key binding for frame-by-frame control.

**Action flag encoding:**
The bitfield pattern (each flag is a power of 2, likely `#define _moving_forward (1<<0)` etc.) allows efficient bitwise accumulation. 20+ flags fit in a 32-bit word; modern engines may use larger action enums or event queues.

## Potential Issues

**1. Single-inclusion constraint is fragile:**
The header comment forbids inclusion by more than `vbl.c`. If preference loading, replay serialization, or Lua scripting need to reference key definitions, they must re-parse or duplicate data. Consider: can this restriction be relaxed, or should key definitions move to a more accessible location?

**2. Undefined backup/override structures:**
`blacklist_data` and `special_flag_data` are declared but never instantiated. If these are intended for features (e.g., preventing "W+A" ghosting, or supporting toggle/latch modifiers), their arrays must exist elsewhereΓÇöbut the cross-reference index doesn't surface them. Risk: dead code, or hard-to-discover feature data scattered across files.

**3. Order-dependency risk:**
Comments state arrays "must be in the same order" for UI dialog alignment. No compile-time check enforces this across multiple layout arrays. If a developer adds a key to `standard_key_definitions` but forgets to update left-handed layout, indices will silently misalign. Consider: enum guards or static assertions.

**4. No version/schema versioning:**
If keyboard layouts are serialized to save files or preferences (likely, given Files subsystem's `wad_prefs.h`), there's no version tag in the struct. Migrating the action flag schema across game versions could silently corrupt saved key bindings.
