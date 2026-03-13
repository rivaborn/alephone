# Source_Files/Misc/progress.h

## File Purpose
Defines the public API for displaying and managing progress dialogs during long-running operations (network transfers, file loading, uploads). Commonly used for blocking operations like map distribution, physics downloads, and Steam Workshop uploads.

## Core Responsibilities
- Define message type constants for various progress operations (network, file I/O, updates)
- Provide dialog lifecycle management (open/close)
- Update and refresh progress dialog messages
- Render and manage progress bar display
- Handle user input events on progress dialogs

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| (unnamed enum) | enum | Message type IDs for progress dialogs; defines `strPROGRESS_MESSAGES` resource ID and operation categories (`_distribute_map_*`, `_receiving_*`, `_loading`, `_uploading_steam_workshop_*`, etc.) |

## Global / File-Static State
None.

## Key Functions / Methods

### open_progress_dialog
- Signature: `void open_progress_dialog(size_t message_id, bool show_progress_bar = false)`
- Purpose: Create and display a progress dialog
- Inputs: `message_id` (enum constant from this file), `show_progress_bar` (whether to render progress bar; defaults to false)
- Outputs/Return: None
- Side effects: Allocates dialog UI resource, displays on screen
- Calls: Not inferable from this file
- Notes: Default parameter suggests progress bar is optional; message_id maps to localized strings

### close_progress_dialog
- Signature: `void close_progress_dialog(void)`
- Purpose: Hide and destroy the progress dialog
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates dialog resource, removes from display
- Calls: Not inferable from this file
- Notes: Inverse of `open_progress_dialog`

### set_progress_dialog_message
- Signature: `void set_progress_dialog_message(size_t message_id)`
- Purpose: Update the message text in an open dialog without closing/reopening it
- Inputs: `message_id` (enum constant)
- Outputs/Return: None
- Side effects: Redraws dialog UI
- Calls: Not inferable from this file
- Notes: Assumes dialog is already open

### draw_progress_bar
- Signature: `void draw_progress_bar(size_t sent, size_t total)`
- Purpose: Render or update progress bar display (e.g., bytes transferred vs. total)
- Inputs: `sent` (current progress), `total` (maximum progress)
- Outputs/Return: None
- Side effects: Updates screen display
- Calls: Not inferable from this file
- Notes: Typically called in frame/draw loop during transfers

### reset_progress_bar
- Signature: `void reset_progress_bar(void)`
- Purpose: Reset progress bar to initial/zero state
- Inputs: None
- Outputs/Return: None
- Side effects: Updates display
- Calls: Not inferable from this file

### progress_dialog_event
- Signature: `void progress_dialog_event(void)`
- Purpose: Handle input events (keyboard, mouse) on the progress dialog
- Inputs: None
- Outputs/Return: None
- Side effects: May dismiss or interact with dialog
- Calls: Not inferable from this file
- Notes: Likely called in main event loop during blocking operations

## Control Flow Notes
Used during blocking operations to display real-time feedback. Typical flow: `open_progress_dialog()` ΓåÆ repeated `draw_progress_bar()` / `set_progress_dialog_message()` calls ΓåÆ `close_progress_dialog()`. Event handling runs in parallel to allow user cancellation or interaction.

## External Dependencies
- `<stddef.h>` ΓÇö `size_t` type only
- Implementation details defined elsewhere (likely `progress.c` or platform-specific files)
