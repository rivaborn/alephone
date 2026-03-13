# Source_Files/Misc/PlayerName.h

## File Purpose
Header file for player name management in netgames. Declares a getter function for the current player name and MML configuration parsing/reset functions for handling default player names.

## Core Responsibilities
- Retrieve the current player name
- Parse player name data from MML configuration trees
- Reset player name configuration to defaults

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| InfoTree | class (forward-declared) | Configuration tree structure for MML parsing |

## Global / File-Static State
None (encapsulated in implementation file).

## Key Functions / Methods

### GetPlayerName
- Signature: `const char *GetPlayerName()`
- Purpose: Retrieve the currently active player name
- Inputs: None
- Outputs/Return: Pointer to player name string (const char*)
- Side effects: None visible
- Calls: (not inferable from header)
- Notes: Likely returns a static or global buffer; caller should not free

### parse_mml_player_name
- Signature: `void parse_mml_player_name(const InfoTree& root)`
- Purpose: Parse player name configuration from an MML data structure
- Inputs: InfoTree reference containing configuration data
- Outputs/Return: None (modifies global player name state)
- Side effects: Updates global player name state
- Calls: (not inferable from header)
- Notes: Called during configuration load phase

### reset_mml_player_name
- Signature: `void reset_mml_player_name()`
- Purpose: Reset player name configuration to default values
- Inputs: None
- Outputs/Return: None
- Side effects: Resets global player name state
- Calls: (not inferable from header)
- Notes: Likely used during cleanup or config reset

## Control Flow Notes
Follows initialization/configuration pattern typical of game engines:
- `parse_mml_player_name()` called during config parsing phase (likely main init or netgame setup)
- `GetPlayerName()` queried at runtime when player name needed
- `reset_mml_player_name()` called when resetting configuration to defaults

## External Dependencies
- `InfoTree` class (declared in another header file)
- GNU General Public License (license context only)
