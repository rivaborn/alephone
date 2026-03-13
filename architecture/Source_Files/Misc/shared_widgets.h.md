# Source_Files/Misc/shared_widgets.h

## File Purpose
Provides preference binding classes and chat history infrastructure for Aleph One dialogs. Bridges strongly-typed C++ preferences to string-based UI widgets via the Bindable pattern, and implements an observer-based chat history system for real-time chat UI updates.

## Core Responsibilities
- Implement `Bindable<T>` subclasses for common preference types (strings, booleans, bit flags, integers, file paths)
- Enable bidirectional data binding between preferences and UI widgets
- Manage chat history with observer notifications
- Adapt chat history to SDL-based colorful chat widget display

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| CStringPref | class | Binds char* C-string buffers to `Bindable<std::string>` |
| BoolPref | class | Binds bool references to `Bindable<bool>` |
| BitPref | class | Binds individual bit flags (uint16 with mask) to `Bindable<bool>` |
| Int16Pref | class | Binds int16 references to `Bindable<int>` |
| FilePref | class | Binds char* file path buffers to `Bindable<FileSpecifier>` |
| ChatHistory | class | Stores vector of chat entries; notifies observer on append/clear |
| ChatHistory::NotificationAdapter | nested class (interface) | Observer interface for chat history updates |
| ColorfulChatWidget | class | Observes `ChatHistory`; forwards updates to `ColorfulChatWidgetImpl` widget |

## Global / File-Static State
None.

## Key Functions / Methods

### CStringPref::bind_export
- Signature: `virtual std::string bind_export()`
- Purpose: Export preference value to UI
- Outputs: std::string copy of C-string buffer
- Side effects: None

### CStringPref::bind_import
- Signature: `virtual void bind_import(std::string s)`
- Purpose: Import UI value back to preference
- Inputs: std::string value
- Side effects: Copies string to bounded char* buffer using `copy_string_to_cstring`

### BitPref::bind_export
- Signature: `virtual bool bind_export()`
- Purpose: Export masked bit flag to UI
- Outputs: bool (bit value, optionally inverted)
- Notes: Applies `m_invert` flag before returning; uses bitwise AND against `m_mask`

### BitPref::bind_import
- Signature: `virtual void bind_import(bool value)`
- Purpose: Import UI boolean back to bit field
- Inputs: bool value
- Side effects: Sets or clears `m_mask` bits in `m_pref`, respecting `m_invert` flag
- Notes: Uses bitwise OR and AND with XOR mask to toggle bits

### FilePref::bind_export
- Signature: `virtual FileSpecifier bind_export()`
- Purpose: Export preference file path to UI
- Outputs: FileSpecifier constructed from C-string buffer
- Notes: Creates temporary FileSpecifier; path encoding handled by FileSpecifier constructor

### FilePref::bind_import
- Signature: `virtual void bind_import(FileSpecifier value)`
- Purpose: Import UI file selection back to preference
- Inputs: FileSpecifier value
- Side effects: Copies path via `strncpy` (max 255 chars) from `value.GetPath()`

### ChatHistory::append
- Signature: `void append(const ColoredChatEntry& e)`
- Purpose: Add chat entry to history and notify observer
- Inputs: ColoredChatEntry (const ref)
- Side effects: Pushes to `m_history` vector; calls `m_notificationAdapter->contentAdded(e)` if observer set

### ChatHistory::setObserver
- Signature: `void setObserver(NotificationAdapter* notificationAdapter)`
- Purpose: Install observer for history updates
- Inputs: NotificationAdapter pointer (may be NULL)
- Side effects: Assigns to `m_notificationAdapter`

### ColorfulChatWidget::attachHistory
- Signature: `void attachHistory(ChatHistory* history)`
- Purpose: Subscribe widget to a ChatHistory's updates
- Inputs: ChatHistory pointer
- Side effects: Stores in `m_history`; registers this widget as observer on the history
- Notes: Caller must ensure history outlives widget

### ColorfulChatWidget::contentAdded
- Signature: `virtual void contentAdded(const ColoredChatEntry& e)`
- Purpose: Callback when history receives new entry
- Inputs: ColoredChatEntry (const ref)
- Side effects: Forwards to `m_componentWidget` (likely appends to UI list)

## Control Flow Notes
This file provides infrastructure for preferenceΓåÆUI and chatΓåÆUI binding, not traditional frame logic. Initialization and updates occur reactively:
1. Preferences are bound via `Bindable<T>` subclasses when dialogs are created
2. ChatHistory receives entries from game/network code; forwards updates to attached widget via observer callback
3. ColorfulChatWidget adapts ChatHistory updates to SDL widget display

## External Dependencies
- **cseries.h**: Provides base types (`uint16`, `int16`, `std::string`), string utilities (`copy_string_to_cstring`)
- **sdl_widgets.h**: Provides `ColorfulChatWidgetImpl` wrapper and `ColoredChatEntry` struct definition
- **binders.h**: Provides `Bindable<T>` template base class
- **FileSpecifier**: Defined elsewhere (likely FileHandler.h); represents file paths with `GetPath()` method
- **STL**: `std::vector`, `std::string` for dynamic storage
