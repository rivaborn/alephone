# Source_Files/Misc/preference_dialogs.h

## File Purpose
Defines an abstract base class for OpenGL preferences dialog with platform-specific implementations. Manages GUI widgets for configuring OpenGL rendering options (texture quality, filtering, visual effects, etc.) through a unified interface.

## Core Responsibilities
- Declare abstract base class for OpenGL preferences dialogs
- Define widget pointers for all configurable OpenGL settings
- Provide abstract methods (`Run`, `Stop`) for platform-specific subclass implementations
- Implement factory pattern (`Create`) to instantiate correct platform-specific dialog at link-time
- Coordinate dialog lifecycle and widget management

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| OpenGLDialog | class | Abstract base class for OpenGL preferences dialog with widget management |

## Global / File-Static State
None.

## Key Functions / Methods

### Create
- Signature: `static std::unique_ptr<OpenGLDialog> Create(int theSelectedRenderer)`
- Purpose: Abstract factory method to instantiate platform-specific concrete dialog class
- Inputs: `theSelectedRenderer` ΓÇö selected OpenGL renderer type
- Outputs/Return: `std::unique_ptr<OpenGLDialog>` ΓÇö ownership-managed dialog instance
- Side effects: None (implementation deferred to link-time)
- Calls: Concrete subclass constructor
- Notes: Concrete implementation determined at link-time; enables compile-time abstraction from platform details (likely SDL vs. Carbon implementations)

### OpenGLPrefsByRunning
- Signature: `void OpenGLPrefsByRunning ()`
- Purpose: Execute the dialog; primary public entry point for showing preferences UI
- Inputs: None
- Outputs/Return: void (side effects only)
- Side effects: Modifies OpenGL preferences state based on user input
- Calls: `Run()` and `Stop()` (virtual methods)
- Notes: Bridges public interface to platform-specific implementation

### Run (pure virtual)
- Signature: `virtual bool Run () = 0`
- Purpose: Platform-specific dialog execution (event loop, widget rendering)
- Outputs/Return: bool (user confirmation result)
- Notes: Subclasses handle all platform UI details (SDL/Carbon)

### Stop (pure virtual)
- Signature: `virtual void Stop (bool result) = 0`
- Purpose: Terminate dialog; apply or discard user changes
- Inputs: `result` ΓÇö whether user confirmed (true) or cancelled (false)
- Notes: Subclasses persist or revert widget state to `OGL_ConfigureData`

## Control Flow Notes
Dialog lifecycle:
1. `Create()` ΓåÆ platform-specific subclass instantiation
2. `OpenGLPrefsByRunning()` ΓåÆ invokes `Run()` (platform event loop)
3. User adjusts widgets (quality, filters, effects, etc.)
4. User confirms/cancels ΓåÆ `Stop(result)` applies or reverts changes
5. Destructor cleanup

Widgets bind to `OGL_ConfigureData` (from `OGL_Setup.h`) via bindable adapters (from `shared_widgets.h`).

## External Dependencies
- **shared_widgets.h** ΓÇö provides widget base classes: `ButtonWidget`, `ToggleWidget`, `SelectorWidget`, `SelectSelectorWidget`, `ColourPickerWidget`; bindable data adapters
- **OGL_Setup.h** ΓÇö provides constants (`OGL_NUMBER_OF_TEXTURE_TYPES`) and configuration structures (`OGL_ConfigureData`); enumerates texture types (wall, landscape, inhabitant, weapons, HUD)
- Standard library: `<memory>` for `std::unique_ptr`
