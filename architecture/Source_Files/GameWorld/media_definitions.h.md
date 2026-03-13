# Source_Files/GameWorld/media_definitions.h

## File Purpose
Defines static configuration data for all media types (water, lava, goo, sewage, Jjaro) in the game world. Each media type specifies visual effects, audio effects, damage properties, and rendering effects when submerged.

## Core Responsibilities
- Define `media_definition` struct for media type configuration
- Initialize static array `media_definitions[NUMBER_OF_MEDIA_TYPES]` with data for all 5 media types
- Specify detonation/splash visual effects for each media size (small, medium, large, emergence)
- Map media sounds (entering, exiting, walking, ambient)
- Define damage properties and frequency for harmful media
- Specify submerged fade/tint effects (color overlay when player underwater)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `media_definition` | struct | Describes all properties of a single media type: visuals, sounds, damage, and effects |
| `damage_definition` | struct | (defined elsewhere) Specifies damage type, amount, and frequency |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `media_definitions` | `media_definition[NUMBER_OF_MEDIA_TYPES]` | static/global | Configuration lookup table for all 5 media types; read-only after init |

## Key Functions / Methods
None ΓÇö this is a pure data definition file.

## Control Flow Notes
This is a configuration/data layer, not part of the runtime loop. The `media_definitions` array is accessed by:
- **Media initialization**: loads visual/audio resources from collection/shape indices
- **Player movement**: looks up damage and ambient effects
- **Effect spawning**: references detonation effects when media is disturbed
- **Rendering**: applies submerged fade effects via `submerged_fade_effect`

## External Dependencies
- **effects.h**: Effect type enumerations (`_effect_small_water_splash`, `_effect_under_water`, etc.)
- **fades.h**: Fade/tint effect types (`_effect_under_water`, `_effect_under_lava`, etc.)
- **media.h**: Media type enums (`_media_water`, `_media_lava`, etc.); `damage_definition` struct
- **SoundManagerEnums.h**: Sound type enumerations (`_snd_enter_water`, `_snd_walking_in_water`, etc.)
