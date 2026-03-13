# Source_Files/Misc/interface_menus.h

## File Purpose
Defines symbolic constants for menu IDs and menu item indices used throughout the game's UI system. Provides centralized declarations of menu identifiers for both in-game menus (pause, save, quit) and main interface menus (new game, preferences, credits).

## Core Responsibilities
- Define menu resource IDs (mGame, mInterface, mFakeEmptyMenu)
- Define menu item indices for game pause menu (iPause, iSave, iRevert, etc.)
- Define menu item indices for main interface menu (iNewGame, iLoadGame, iPreferences, etc.)
- Provide constants to decouple menu resource numbers from UI handling code

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| (unnamed enum) | enum | Game menu items (ID 128): pause, save, revert, close, quit |
| (unnamed enum) | enum | Interface menu items (ID 129): new game, load, gather, join, preferences, replay, credits, quit, singleton level, about |
| (unnamed enum) | enum | Fake empty menu (ID 130): placeholder to prevent menu bar display on exit |

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
Not applicableΓÇöthis is a pure constants header. Menu IDs and item indices are used by menu event handling code elsewhere to dispatch user selections (presumably during frame updates and input handling).

## External Dependencies
- Standard C header guards (`#ifndef`, `#define`, `#endif`)
- No external includes or symbol dependencies
