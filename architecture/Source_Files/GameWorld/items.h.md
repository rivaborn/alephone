# Source_Files/GameWorld/items.h

## File Purpose
Header file defining the item system for the Aleph One game engine (Marathon-based). Declares enumerations for item types and categories, and provides prototypes for item creation, inventory management, and XML configuration parsing.

## Core Responsibilities
- Define item type enumerations (weapon, ammunition, powerup, ball, etc.)
- Declare item lifecycle functions (creation, animation, initialization)
- Declare inventory/player item interaction functions
- Declare item query utilities (shape, validity, kind)
- Declare XML/MML configuration parsing for item customization

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| (anonymous enum at line 48) | enum | Item class types: weapon, ammunition, powerup, ball, etc. |
| (anonymous enum at line 56) | enum | Specific item IDs (knife, magnum, plasma rifle, powerups, balls, etc.) |
| `object_location` | struct | Referenced but defined elsewhere; location data for spawning items |
| `InfoTree` | class | Referenced for XML/MML parsing; defined elsewhere |

## Global / File-Static State
None.

## Key Functions / Methods

### new_item
- Signature: `short new_item(struct object_location *location, short item_type);`
- Purpose: Create a new item instance at a given location.
- Inputs: location pointer, item type ID.
- Outputs/Return: Item ID or error code (short).
- Side effects: Modifies game world state; allocates item object.

### calculate_player_item_array
- Signature: `void calculate_player_item_array(short player_index, short type, short *items, short *counts, short *array_count);`
- Purpose: Query player's inventory; retrieve items of a specific type with counts.
- Inputs: Player index, item type filter.
- Outputs/Return: Populates items, counts, and array_count pointers.
- Side effects: None.

### try_and_add_player_item
- Signature: `bool try_and_add_player_item(short player_index, short type);`
- Purpose: Attempt to add an item to a player's inventory.
- Inputs: Player index, item type.
- Outputs/Return: Boolean success/failure.
- Side effects: Modifies player inventory on success.

### find_player_ball_color
- Signature: `short find_player_ball_color(short player_index);`
- Purpose: Determine the color of the ball (if any) carried by a player; used in King of the Hill/CTF.
- Inputs: Player index.
- Outputs/Return: Ball color ID or NONE.
- Side effects: None.

### initialize_items & animate_items
- Signatures: `void initialize_items(void);` and `void animate_items(void);`
- Purpose: Setup item system at startup; update animations each frame.
- Calls: Likely called from main game loop and initialization.
- Side effects: Modifies item animation state.

### parse_mml_items
- Signature: `void parse_mml_items(const InfoTree& root);`
- Purpose: Parse XML/MML configuration for item customization (weapons, powerups, spawn rates, etc.).
- Inputs: InfoTree reference (XML DOM-like structure).
- Outputs/Return: Void.
- Side effects: Modifies global item configuration.

## Control Flow Notes
- **Initialization**: `initialize_items()` called during game startup.
- **Per-frame**: `animate_items()` called each frame to update item animations and visuals.
- **Inventory**: `calculate_player_item_array()` and `try_and_add_player_item()` used when players pick up or use items.
- **Spawning**: `new_item()` or `new_item_in_random_location()` called by map triggers or gameplay logic.
- **Configuration**: `parse_mml_items()` processes custom item definitions from XML (post-2000 addition).

## External Dependencies
- `object_location` ΓÇö struct for world coordinates/orientation (defined elsewhere).
- `InfoTree` ΓÇö XML parsing class (MML/XML configuration format).
- References to player and polygon systems (CTF ball, triggers).

**Notes**: This is a pure declaration header; implementation is in `items.c`. The file evolved significantly (Feb 2000ΓÇôMay 2000) to add SMG weapons, XML customization, and animator/initializer patterns borrowed from `scenery.h`.
