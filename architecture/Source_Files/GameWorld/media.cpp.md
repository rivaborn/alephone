# Source_Files/GameWorld/media.cpp

## File Purpose
Implements management of in-world liquids/media (water, lava, goo, sewage, jjaro). Handles instantiation, per-frame updates, property queries, serialization, and XML-based configuration of liquid behavior and effects.

## Core Responsibilities
- Create and maintain media instances (liquids) in the game world
- Calculate and update media heights based on dynamic light intensity
- Retrieve media-specific properties (damage, sounds, effects, textures, fade effects)
- Verify media availability in specific environment types
- Serialize/deserialize media state for save/load
- Parse and apply XML-based liquid configuration (MML)
- Evaluate whether media is dangerous to entities

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `media_data` | struct | Runtime instance of a liquid; tracks position, height, texture, flow parameters (32 bytes) |
| `media_definition` | struct | Template for a media type; stores collection, shape, damage, sounds, detonation effects (defined in media_definitions.h) |
| `damage_definition` | struct | Damage properties for media (from map.h); includes type, base damage, frequency |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MediaList` | `vector<media_data>` | global | Dynamic array of all active media instances in current level |
| `media_definitions` | `media_definition[]` | global (from media_definitions.h) | Array of pre-defined media type templates |
| `original_media_definitions` | `media_definition*` | static | Backup of unmodified definitions for MML reset |

## Key Functions / Methods

### get_media_data
- Signature: `media_data *get_media_data(const size_t media_index)`
- Purpose: Safe accessor to retrieve media instance by index
- Inputs: media_index (validated against bounds and slot usage)
- Outputs/Return: `media_data*` or NULL if invalid
- Side effects: None
- Calls: GetMemberWithBounds macro, SLOT_IS_USED macro
- Notes: Dual validation (bounds + slot-in-use); returns NULL on either failure

### new_media
- Signature: `size_t new_media(struct media_data *initializer)`
- Purpose: Instantiate a new media (liquid) in the world
- Inputs: initializer containing media properties (type, flags, light_index, etc.); light_index must be pre-loaded
- Outputs/Return: media_index (0ΓÇôMAXIMUM_MEDIAS_PER_MAPΓÇô1) or UNONE if table full
- Side effects: Marks slot used, initializes origin to (0,0), calls update_one_media
- Calls: update_one_media
- Notes: Finds first free slot; invariant: light_index must be valid at call time

### update_medias
- Signature: `void update_medias(void)`
- Purpose: Per-frame update of all active media (heights, positions, textures)
- Inputs: None
- Outputs/Return: None
- Side effects: Updates height, texture, transfer_mode, and applies flow (updates origin based on current_direction/current_magnitude and trig tables)
- Calls: update_one_media
- Notes: Uses fixed-point trigonometry (trig_shift) and wrapping arithmetic (WORLD_FRACTIONAL_PART) for flow calculation

### update_one_media
- Signature: `void update_one_media(size_t media_index, bool force_update)`
- Purpose: Recalculate height and visual properties for a single media instance
- Inputs: media_index, force_update (currently unused)
- Outputs/Return: None
- Side effects: Updates media->height (computed from light intensity), texture, transfer_mode
- Calls: get_media_data, get_media_definition, get_light_intensity
- Notes: Height is dynamic: `low + (highΓêÆlow) ├ù light_intensity`; clever use of light system to drive liquid surface animation

### get_media_detonation_effect, get_media_sound, get_media_damage, get_media_submerged_fade_effect, get_media_collection
- Pattern: All follow similar safe-accessor idiom
- Purpose: Retrieve media-type-specific properties (detonation effects, ambient/interaction sounds, damage on contact, submerged visual fade, texture collection)
- Inputs: media_index (or media_type) and optional type parameter (effect/sound classification)
- Outputs/Return: Scalar/pointer property or NULL/NONE if invalid
- Side effects: None (get_media_damage modifies damage.scale in-place)
- Calls: get_media_data, get_media_definition
- Notes: All validate that media and definition exist before returning

### IsMediaDangerous
- Signature: `bool IsMediaDangerous(short media_index)`
- Purpose: Determine if a media type inflicts damage
- Inputs: media_index (actually media *type*, not instance index; parameter naming inconsistent)
- Outputs/Return: bool (true if media.damage.base > 0)
- Side effects: None
- Calls: get_media_definition
- Notes: **Semantic issue**: signature uses `short` and name suggests instance index, but function treats input as media type (enum value)

### pack_media_data, unpack_media_data
- Signature: `uint8 *pack_media_data(uint8 *Stream, media_data* Objects, size_t Count)` and reverse
- Purpose: Serialize/deserialize media array to/from binary stream (for save/load)
- Inputs: Stream (byte buffer), Objects (array), Count (number of instances)
- Outputs/Return: Updated Stream pointer (post-serialization position)
- Side effects: Modifies stream buffer; on unpack, modifies Objects array
- Calls: ValueToStream / StreamToValue macros (from Packing.h)
- Notes: Loops over Count objects; writes 14 fields per object plus 2-byte padding; asserts stream position matches expected size

### parse_mml_liquids
- Signature: `void parse_mml_liquids(const InfoTree& root)`
- Purpose: Load and apply XML-based liquid configuration (MML modding system)
- Inputs: InfoTree root (parsed XML document)
- Outputs/Return: None
- Side effects: Modifies media_definitions array; backs up originals if not already done
- Calls: InfoTree methods (read_indexed, read_attr, children_named), read_damage
- Notes: Backs up original definitions on first call; iterates over `<liquid>` elements; parses nested `<sound>`, `<effect>`, `<damage>` sub-elements; supports partial/incremental updates

### reset_mml_liquids
- Signature: `void reset_mml_liquids()`
- Purpose: Restore media definitions to pre-MML state
- Inputs: None
- Outputs/Return: None
- Side effects: Restores media_definitions from original_media_definitions backup; frees backup
- Calls: free()
- Notes: Safe to call multiple times (checks if backup exists)

### count_number_of_medias_used
- Signature: `size_t count_number_of_medias_used()`
- Purpose: Count how many media slots are in active use (for save-game compatibility with Infinity engine)
- Inputs: None
- Outputs/Return: Highest index + 1 where SLOT_IS_USED is true, or 0 if none used
- Side effects: None
- Calls: SLOT_IS_USED macro
- Notes: Iterates backwards; returns first (highest) used slot; fixed countdown bug parallel to map.cpp

### media_in_environment
- Signature: `bool media_in_environment(short media_type, short environment_code)`
- Purpose: Check if a media type is available/supported in a specific environment
- Inputs: media_type, environment_code
- Outputs/Return: bool
- Side effects: None
- Calls: get_media_definition, collection_in_environment
- Notes: Idiot-proofing: returns false if definition not found

## Control Flow Notes
**Initialization:** parse_mml_liquids() loads XML config; media_definitions populated  
**Level Setup:** new_media() places liquid areas in polygons  
**Per-Frame:** update_medias() called to animate heights and flow; height drives visual effects (via light intensity)  
**Property Queries:** Systems (rendering, physics, audio, effects) call get_media_* accessors during gameplay  
**Serialization:** pack/unpack during save/load transitions  
**Cleanup:** reset_mml_liquids() restores defaults on config reload or shutdown

## External Dependencies
- **map.h**: SLOT_IS_USED macro, damage_definition struct, dynamic_world, shape_descriptor, world types
- **effects.h**: Effect type enums (NUMBER_OF_EFFECT_TYPES)
- **fades.h**: Fade effect enums (NUMBER_OF_FADE_EFFECT_TYPES)
- **lightsource.h**: get_light_intensity() function
- **SoundManager.h**: (included but not directly used in this file)
- **InfoTree.h**: XML parsing class for MML support
- **Packing.h**: StreamToValue, ValueToStream macros for serialization
- **media_definitions.h** (not bundled): media_definition struct and media_definitions array
- **cseries.h**: Platform types, macros (FIXED_INTEGERAL_PART, WORLD_FRACTIONAL_PART, TRIG_SHIFT)
- **cmath functions**: trig tables (cosine_table, sine_table) assumed global
