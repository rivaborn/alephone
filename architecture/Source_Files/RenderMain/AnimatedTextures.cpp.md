# Source_Files/RenderMain/AnimatedTextures.cpp

## File Purpose
Implements animated wall textures for the Aleph One game engine. Manages frame sequences that cycle through texture indices at configurable rates, supporting bidirectional animation. Animations are stored per collection and applied during texture lookup via XML configuration.

## Core Responsibilities
- Store and manage animation sequences (frame lists) indexed by collection
- Update animation state (frame and tick phases) each frame
- Translate texture descriptors to their current animated frame
- Parse XML configuration to create and configure animation objects
- Support selective texture animation (animate specific texture or all in list)
- Support bidirectional animation (positive/negative tick rates)
- Clear and reset animation data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `AnimTxtr` | class | Single animation sequence with frame list, timing parameters, and phase tracking |
| `AnimTxtrList` | static array | Collection of `AnimTxtr` objects, one vector per collection ID |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AnimTxtrList` | `vector<AnimTxtr>[NUMBER_OF_COLLECTIONS]` | static | All animation sequences indexed by collection; enables O(1) collection lookup |

## Key Functions / Methods

### AnimTxtr::Load
- Signature: `void Load(vector<short>& _FrameList)`
- Purpose: Initialize animation with a list of texture frame indices
- Inputs: Vector of frame IDs (by reference, consumed via swap)
- Outputs/Return: None
- Side effects: Swaps internal frame list; wraps `FramePhase` if out of range
- Calls: `vector::swap()`, `vector::empty()`, modulo operator
- Notes: Uses move semantics (swap); does not copy frame data

### AnimTxtr::Translate
- Signature: `bool Translate(short& Frame)`
- Purpose: Map a texture descriptor to its current animated frame index
- Inputs: Frame ID (by reference, may be modified in-place)
- Outputs/Return: `true` if frame was translated, `false` if no match found
- Side effects: Modifies input `Frame` in-place
- Calls: Direct comparison; modulo operator
- Notes: If `Select >= 0`, only translates matching frame; else searches whole list; offsets frame index by `FramePhase`

### AnimTxtr::SetTiming
- Signature: `void SetTiming(short _NumTicks, size_t _FramePhase, size_t _TickPhase)`
- Purpose: Configure animation timing and initial phase
- Inputs: Ticks per frame (negative = reverse), frame phase, tick phase
- Outputs/Return: None
- Side effects: Updates timing parameters; normalizes tick/frame phases to valid ranges
- Calls: None (inline arithmetic)
- Notes: Handles phase wrap-around; corrects `TickPhase` overflow into `FramePhase`

### AnimTxtr::Update
- Signature: `void Update()`
- Purpose: Advance animation state by one tick
- Inputs: None
- Outputs/Return: None
- Side effects: Increments `TickPhase` and `FramePhase` with wraparound; direction depends on `NumTicks` sign
- Calls: None
- Notes: Handles forward (NumTicks > 0) and reverse (NumTicks < 0) animation separately; no-op if `NumTicks == 0`

### AnimTxtr_Update
- Signature: `void AnimTxtr_Update()`
- Purpose: Update all animations in all collections
- Inputs: None
- Outputs/Return: None
- Side effects: Calls `Update()` on every `AnimTxtr` instance
- Calls: `AnimTxtr::Update()`
- Notes: Called once per frame; iterates nested loops: collections ΓåÆ animation sequences

### AnimTxtr_Translate
- Signature: `shape_descriptor AnimTxtr_Translate(shape_descriptor Texture)`
- Purpose: Apply animation to a texture descriptor, returning the current animated frame
- Inputs: Shape descriptor (encodes collection, frame, color table)
- Outputs/Return: Updated shape descriptor with animated frame
- Side effects: None (pure function on descriptor state)
- Calls: `AnimTxtr::Translate()`, `get_number_of_collection_frames()` (validation), descriptor macros
- Notes: Returns `UNONE` if descriptor is `UNONE`, frame is out of range, or no animation matches; iterates animation list for collection

### parse_mml_animated_textures
- Signature: `void parse_mml_animated_textures(const InfoTree& root)`
- Purpose: Parse XML configuration and populate animation sequences
- Inputs: InfoTree root node (XML tree)
- Outputs/Return: None
- Side effects: Creates and appends `AnimTxtr` objects to `AnimTxtrList[coll]`
- Calls: `InfoTree::read_indexed()`, `InfoTree::read_attr()`, `InfoTree::children_named()`, `AnimTxtr::Load()`, `AnimTxtr::SetTiming()`
- Notes: Supports two XML node types: "clear" (delete collection or all) and "sequence" (create animation); skips malformed sequences

### reset_mml_animated_textures
- Signature: `void reset_mml_animated_textures()`
- Purpose: Clear all animations (used before reloading XML config)
- Inputs: None
- Outputs/Return: None
- Side effects: Calls `ATDeleteAll()`
- Calls: `ATDeleteAll()`
- Notes: Trivial wrapper; enables symmetry with `parse_mml_animated_textures()`

## Control Flow Notes
- **Initialization**: `parse_mml_animated_textures()` called at startup/config reload to populate `AnimTxtrList` from XML
- **Per-frame update**: `AnimTxtr_Update()` called each frame to advance all animation phases
- **Texture lookup**: `AnimTxtr_Translate()` called during rendering to map descriptors to current animated frames
- **Cleanup**: `reset_mml_animated_textures()` called before reconfiguration to clear state

## External Dependencies
- **Standard Library**: `<vector>`, `<string.h>`
- **Engine**: `cseries.h` (base types, macros), `interface.h` (shape descriptor macros, `get_number_of_collection_frames()`), `InfoTree.h` (XML parsing)
- **Macros defined elsewhere**: `GET_DESCRIPTOR_SHAPE()`, `GET_DESCRIPTOR_COLLECTION()`, `GET_COLLECTION()`, `GET_COLLECTION_CLUT()`, `BUILD_COLLECTION()`, `BUILD_DESCRIPTOR()`, `UNONE`, `NUMBER_OF_COLLECTIONS`
