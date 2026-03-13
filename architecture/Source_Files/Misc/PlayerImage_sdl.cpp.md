# Source_Files/Misc/PlayerImage_sdl.cpp

## File Purpose
Implements SDL-based rendering for player character images in Aleph One. Manages loading, caching, and compositing of 2D player sprites consisting of separate legs and torso parts with configurable appearance (view angle, color, action, animation frame, brightness).

## Core Responsibilities
- Load and cache SDL surfaces for player legs and torso from shape collections
- Resolve appearance parameters (action, view, frame, color) with random fallback and retry logic
- Composite and render legs + torso sprites to a target SDL surface at specified screen coordinates
- Manage collection lifecycle via reference counting (load on first object creation, unload on last destruction)
- Validate appearance parameters and degrade gracefully if invalid or if rendering data cannot be loaded

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| PlayerImage | class (external) | Player character sprite renderer; aggregates state (view, color, action, frame) and cached drawing data (surfaces, rects) |
| shape_animation_data | struct (external) | Animation frame metadata: view count, frames per view, frame indices into low-level shapes |
| low_level_shape_definition | struct (external) | Individual sprite metadata: origin/key point offsets (pixel coordinates), world bounds |
| SDL_Surface | struct (external) | SDL pixel buffer; points to sprite bitmap data |
| SDL_Rect | struct (external) | SDL rectangle; used for positioning sprites during blit |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| bit_depth | extern short | global | Rendering bit depth; 8-bit is skipped (early continue in shape loading) |
| sNumOutstandingObjects | static int16 | class-static | Active PlayerImage instance counter; drives collection load/unload |

## Key Functions / Methods

### `~PlayerImage()`
- Signature: `PlayerImage::~PlayerImage()`
- Purpose: Clean up allocated resources when instance is destroyed
- Inputs: None (implicit this)
- Outputs/Return: None
- Side effects: Frees SDL_Surface objects (mLegsSurface, mTorsoSurface) via SDL_FreeSurface; frees pixel buffers (mLegsData, mTorsoData) via free(); calls objectDestroyed() to decrement reference count
- Calls: SDL_FreeSurface, free, objectDestroyed
- Notes: Defensive NULL checks before each free

### `updateLegsDrawingInfo()`
- Signature: `void updateLegsDrawingInfo()`
- Purpose: Resolve legs appearance parameters and load the corresponding sprite surface from the shape collection
- Inputs: None (uses member variables mLegsAction, mLegsView, mLegsFrame, mLegsColor)
- Outputs/Return: None
- Side effects: Frees old surfaces/data; loads new mLegsSurface and mLegsData via get_shape_surface; updates mLegsRect positioning; sets mLegsValid and clears mLegsDirty
- Calls: SDL_FreeSurface, free, get_player_shape_definitions, get_shape_animation_data, local_random, get_low_level_shape_definition, get_shape_surface
- Notes: 
  - Retry loop (up to 100 attempts): NONE parameters pick random values; failed loads trigger retry with new random choices
  - User-supplied out-of-range parameters cause early exit with invalid data (no retry)
  - Skips 8-bit color mode with early continue
  - Computes low-level shape index as `frames_per_view * view + frame` from animation metadata

### `updateTorsoDrawingInfo()`
- Signature: `void updateTorsoDrawingInfo()`
- Purpose: Resolve torso appearance parameters and load the corresponding sprite surface
- Inputs: None (uses member variables mTorsoAction, mPseudoWeapon, mTorsoView, mTorsoFrame, mTorsoColor)
- Outputs/Return: None
- Side effects: Same as updateLegsDrawingInfo but for torso
- Calls: SDL_FreeSurface, free, get_player_shape_definitions, get_shape_animation_data, local_random, get_low_level_shape_definition, get_shape_surface
- Notes: 
  - Similar retry logic and parameter validation as legs
  - Selects torso array (firing_torsos, torsos, or charging_torsos) via switch on mTorsoAction
  - Uses origin_x/origin_y for positioning (vs. key_x/key_y for legs)

### `drawAt(SDL_Surface* inSurface, int16 inX, int16 inY)`
- Signature: `void drawAt(SDL_Surface* inSurface, int16 inX, int16 inY)`
- Purpose: Render the player image (legs + torso) to a target SDL surface at screen coordinates
- Inputs: inSurface (target SDL surface), inX, inY (screen position)
- Outputs/Return: None
- Side effects: Modifies inSurface by blitting mLegsSurface and mTorsoSurface at offset positions
- Calls: canDrawLegs, canDrawTorso, SDL_BlitSurface
- Notes: 
  - canDrawLegs/canDrawTorso implicitly trigger updateDrawingInfo() if dirty
  - Only renders legs/torso if valid (data was successfully loaded)
  - Adjusts rect positions by inX/inY screen offset and rect's origin offsets

### `objectCreated()`
- Signature: `static void objectCreated()`
- Purpose: Register a new PlayerImage instance; load player shape collection on first instance
- Inputs: None
- Outputs/Return: None
- Side effects: Increments sNumOutstandingObjects; if count transitions from 0ΓåÆ1, marks and loads the player shape collection via mark_collection and load_collections
- Calls: get_player_shape_definitions, mark_collection, load_collections
- Notes: Commented-out code hints at a prior workaround for duplicate loading

### `objectDestroyed()`
- Signature: `static void objectDestroyed()`
- Purpose: Unregister a PlayerImage instance; unload player shape collection when last instance is destroyed
- Inputs: None
- Outputs/Return: None
- Side effects: Decrements sNumOutstandingObjects; if count reaches 0, unmarks the player shape collection
- Calls: get_player_shape_definitions, mark_collection
- Notes: Simple reference counting to coordinate collection lifecycle

## Control Flow Notes
**Initialization:** Constructor calls objectCreated() to ensure the player shape collection is resident.

**Dirty-flag update cycle:** Setters (inherited from class) mark mLegsDirty/mTorsoDirty when parameters change. No immediate work is done.

**Lazy loading:** drawAt() or canDrawLegs/canDrawTorso/canDrawPlayer queries trigger updateDrawingInfo(), which calls updateLegsDrawingInfo/updateTorsoDrawingInfo as needed. These resolve random NONE parameters, validate ranges, and load SDL surfaces from the shape collection.

**Rendering:** drawAt() blits cached surfaces to the target surface at screen coordinates, adjusted by sprite origin offsets.

**Cleanup:** ~PlayerImage() frees surfaces and calls objectDestroyed() to unload the collection if no instances remain.

## External Dependencies
- **SDL:** SDL_Surface, SDL_FreeSurface, SDL_BlitSurface, SDL_Rect
- **Engine shape/animation:** get_player_shape_definitions, get_shape_animation_data, get_low_level_shape_definition, get_shape_surface (from shell.h, interface.h)
- **Engine math/random:** local_random (from world.h)
- **Engine constants:** NUMBER_OF_PLAYER_ACTIONS, PLAYER_TORSO_WEAPON_ACTION_COUNT, PLAYER_TORSO_SHAPE_COUNT, _shape_weapon_* action enums (from player.h, weapons.h)
- **Engine collection/shape descriptors:** BUILD_DESCRIPTOR, BUILD_COLLECTION macros, low_level_shape_definition, shape_animation_data (from collection_definition.h, interface.h)
- **Global state:** extern bit_depth (color depth indicator)
