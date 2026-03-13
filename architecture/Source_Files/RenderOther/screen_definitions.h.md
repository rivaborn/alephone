# Source_Files/RenderOther/screen_definitions.h

## File Purpose
Defines resource ID base constants for all major UI/screen elements in the game. Establishes a naming convention where 8-bit picture IDs are base values, 16-bit versions add 10000, and 32-bit versions add 20000.

## Core Responsibilities
- Define base resource IDs for intro, menu, prologue, epilogue, credit, chapter, computer interface, panel, and final screens
- Document the bit-depth offset scheme (8-bit + 0, 16-bit + 10000, 32-bit + 20000)
- Provide a centralized enumeration for screen resource lookups across the rendering and UI subsystems

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| INTRO_SCREEN_BASE | enum constant | global | Base ID (1000) for intro screen graphics resources |
| MAIN_MENU_BASE | enum constant | global | Base ID (1100) for main menu resources |
| PROLOGUE_SCREEN_BASE | enum constant | global | Base ID (1200) for prologue screen resources |
| EPILOGUE_SCREEN_BASE | enum constant | global | Base ID (1300) for epilogue screen resources |
| CREDIT_SCREEN_BASE | enum constant | global | Base ID (1400) for credits screen resources |
| CHAPTER_SCREEN_BASE | enum constant | global | Base ID (1500) for chapter/level intro resources |
| COMPUTER_INTERFACE_BASE | enum constant | global | Base ID (1600) for computer/terminal UI resources |
| INTERFACE_PANEL_BASE | enum constant | global | Base ID (1700) for HUD/panel UI resources |
| FINAL_SCREEN_BASE | enum constant | global | Base ID (1800) for end-game/final screen resources |

## Key Functions / Methods
None.

## Control Flow Notes
No control flow. This is a pure definition header with no executable code. Used at resource-load time to map logical screen types to their picture resource IDs in the resource fork (Classic Mac) or asset store.

## External Dependencies
- `<stdio.h>` implicit (standard C header conventions)
- No other includes visible
