# Source_Files/Misc/PlayerImage_sdl.h

## File Purpose
Defines a C++ class for rendering and managing sprite images of player characters in Aleph One (Marathon engine). Handles separate legs/torso rendering with configurable state (view angle, color, action, animation frame, brightness) and lazy-loads expensive image data on demand.

## Core Responsibilities
- Manage player character visual state: view direction, team/player colors, leg/torso actions, animation frames, brightness, size
- Lazy-load and cache SDL surfaces for leg and torso graphics only when needed
- Mark/unmark image collections for memory management via reference counting
- Provide status inquiry methods to check whether valid drawing data is available
- Render player image at specified screen coordinates with current state

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| PlayerImage | class | Main class managing player sprite state and rendering |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| sNumOutstandingObjects | int16 | static | Reference count of active PlayerImage instances; used to mark/unmark SDL collections |

## Key Functions / Methods

### Constructor (default)
- Signature: `PlayerImage()`
- Purpose: Initialize with neutral/random state; calls `objectCreated()`
- Inputs: None
- Outputs/Return: Initialized object with NONE values and NULL surfaces
- Side effects: Increments `sNumOutstandingObjects`; may mark collections
- Notes: Lazy initializationΓÇöexpensive image fetching deferred until first draw/update

### Destructor
- Signature: `~PlayerImage()`
- Purpose: Clean up allocated surfaces and data buffers
- Side effects: Decrements `sNumOutstandingObjects`; may unmark collections; frees SDL surfaces and byte arrays
- Calls: `objectDestroyed()`

### setLegsView / setTorsoView / setView
- Signature: `void setLegsView(int16 inView)`, etc.
- Purpose: Set view direction (0ΓÇô7; 0 = facing viewer, increments rotate 45┬░ right)
- Inputs: View index (NONE = pick randomly)
- Side effects: Sets dirty flag if value changed
- Notes: NONE (-1) triggers random selection; no re-fetch until draw

### setLegsColor / setTorsoColor / setColor
- Signature: `void setLegsColor(int16 inColor)`, etc.
- Purpose: Set color palette index (0ΓÇô7; standard team/player colors)
- Inputs: Color index (NONE = random)
- Side effects: Sets dirty flag if changed

### setLegsAction / setTorsoAction
- Signature: `void setLegsAction(int16 inAction)`, etc.
- Purpose: Set animation action (from `player.h`, `weapons.h` enums; e.g., `_player_running`)
- Inputs: Action enum value or NONE
- Side effects: Sets dirty flag

### setPseudoWeapon
- Signature: `void setPseudoWeapon(int16 inWeapon)`
- Purpose: Set weapon torso should appear to hold (enum from `weapons.h`)
- Inputs: Weapon enum or NONE
- Side effects: Sets torso dirty flag

### setLegsFrame / setTorsoFrame / setRandomFrame
- Signature: `void setLegsFrame(int16 inFrame)`, etc.
- Purpose: Set animation frame index (0ΓÇômax; depends on action)
- Inputs: Frame number or NONE
- Side effects: Sets dirty flag

### setLegsBrightness / setTorsoBrightness / setBrightness
- Signature: `void setLegsBrightness(float inBrightness)`, etc.
- Purpose: Set brightness (0.0ΓÇô1.0)
- Inputs: Float brightness value
- Side effects: Sets dirty flag

### setTiny
- Signature: `void setTiny(bool inTiny)`
- Purpose: Enable/disable quarter-sized rendering
- Inputs: Boolean
- Side effects: Sets both legs and torso dirty flags

### updateLegsDrawingInfo / updateTorsoDrawingInfo / updateDrawingInfo
- Signature: `void updateDrawingInfo()`
- Purpose: Fetch and cache SDL surfaces and rects from image collections if dirty flags are set
- Side effects: Populates `mLegsSurface`, `mTorsoSurface`, `mLegsData`, `mTorsoData`, rects; clears dirty flags; sets validity flags
- Notes: Re-picks random values if fetch fails; repeated failures leave validity false

### canDrawLegs / canDrawTorso / canDrawSomething / canDrawPlayer
- Signature: `bool canDrawLegs()`, etc.
- Purpose: Check whether valid drawing data is available; calls `updateDrawingInfo()` first
- Outputs/Return: Boolean validity status
- Notes: Aggregators: `canDrawSomething()` = legs OR torso; `canDrawPlayer()` = legs AND torso

### getTeam
- Signature: `int16 getTeam()`
- Purpose: Query team color after ensuring data is up-to-date
- Outputs/Return: Team color index
- Calls: `updateDrawingInfo()`

### drawAt
- Signature: `void drawAt(SDL_Surface* inSurface, int16 inX, int16 inY)`
- Purpose: Render player image to target SDL surface at given coordinates
- Inputs: Target surface, screen X/Y position
- Side effects: Calls `updateDrawingInfo()` first; blits legs/torso rects to target
- Notes: Only draws valid parts (legs, torso, or both)

## Control Flow Notes
**Pattern: Lazy validation & dirty-flag rendering**

1. **Initialization**: Constructor calls `objectCreated()` (marks collections)
2. **State modification**: Setters mark legs/torso dirty if value changes; no immediate fetch
3. **Pre-render sync**: `updateDrawingInfo()` fetches SDL surfaces only if dirty; resolves NONE values; sets validity flags
4. **Rendering**: `drawAt()` calls update, then blits to target surface
5. **Teardown**: Destructor frees surfaces/data and calls `objectDestroyed()` (unmarks collections when count reaches 0)

This enables efficient prefetching and batches expensive image loads with the frame that needs them.

## External Dependencies
- **SDL2**: `SDL_Surface`, `SDL_Rect` (graphics surfaces and rectangles)
- **cseries.h**: Type definitions (`int16`, `byte`), SDL includes, cross-platform macros
- **Implicit external symbols** (not defined here):
  - `player.h`, `weapons.h` ΓÇö action and weapon enum constants
  - Collection/image loading functions (called within `updateLegsDrawingInfo()`, `updateTorsoDrawingInfo()`)
  - `objectCreated()`, `objectDestroyed()` implementations
