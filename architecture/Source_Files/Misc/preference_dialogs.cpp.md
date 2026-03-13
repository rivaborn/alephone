# Source_Files/Misc/preference_dialogs.cpp

## File Purpose
Implements OpenGL graphics preferences dialog functionality for the Aleph One game engine. Provides bindable preference adapter classes that translate between internal preference values and UI widgets, and implements both abstract dialog interface and concrete SDL-based dialog UI.

## Core Responsibilities
- Define preference binder classes that convert between preference data types and UI-displayable forms (e.g., texture size Γåö quality index)
- Manage lifecycle of OpenGL graphics preferences dialog (construction, running, applying changes)
- Build hierarchical dialog UI with tabs, tables, toggles, sliders, and dropdown selectors for graphics settings
- Coordinate data binding between UI widgets and graphics preference structures via `BinderSet`
- Persist user-modified preferences back to disk when dialog is accepted

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| TexQualityPref | class | Adapts texture max-size preference to/from quality UI index using logarithmic scale |
| ColourPref | class | Direct passthrough adapter for RGBColor preference binding |
| FarFilterPref | class | Maps preference filter enum (3, 5) to UI indices (2, 3) |
| TimesTwoPref | class | Divides/multiplies preference by 2 for UI display |
| AnisotropyPref | class | Converts float anisotropy level to/from power-of-2 bit-index UI representation |
| w_aniso_slider | class | Custom slider widget that displays anisotropy as powers of 2 (1, 2, 4, 8, etc.) |
| OpenGLDialog | class | Abstract base class defining preferences dialog interface |
| SdlOpenGLDialog | class | Concrete SDL/UI implementation; builds and manages dialog UI hierarchy |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| far_filter_labels | const char*[5] | file-static | Label strings for far-distance texture filters (None, Linear, Bilinear, Trilinear) |
| near_filter_labels | const char*[3] | file-static | Label strings for near-distance texture filters (None, Linear) |
| ephemera_quality_labels | std::vector\<std::string\> | file-static | Labels for scripted effects quality levels (Off, Low, Medium, High, Ultra) |

## Key Functions / Methods

### TexQualityPref::bind_export()
- Signature: `int bind_export()`
- Purpose: Convert internal texture size preference to UI quality index
- Inputs: none (uses `m_pref` and `m_normal`)
- Outputs/Return: int index where index = log2(m_pref / m_normal), clamped at 0
- Side effects: none
- Calls: none
- Notes: Performs right-shift loop to compute log2; handles zero case

### TexQualityPref::bind_import()
- Signature: `void bind_import(int value)`
- Purpose: Convert UI quality index back to texture size preference
- Inputs: int value (0 = unlimited/0, 1 = normal, 2 = normal/2, etc.)
- Outputs/Return: none (modifies `m_pref`)
- Side effects: Updates `m_pref`
- Calls: none
- Notes: value=0 ΓåÆ 0; otherwise 2^(value-1) * m_normal

### FarFilterPref::bind_export()
- Signature: `int bind_export()`
- Purpose: Translate stored filter enum value to UI index
- Inputs: none (uses `m_pref`)
- Outputs/Return: int UI index (maps 5ΓåÆ3, 3ΓåÆ2, others unchanged)
- Side effects: none
- Calls: none
- Notes: Handles legacy/remapped filter values

### FarFilterPref::bind_import()
- Signature: `void bind_import(int value)`
- Purpose: Translate UI index back to filter enum
- Inputs: int value (UI selection index)
- Outputs/Return: none (modifies `m_pref`)
- Side effects: Updates `m_pref` (maps 3ΓåÆ5, 2ΓåÆ3, others unchanged)
- Calls: none
- Notes: Reverse of bind_export()

### AnisotropyPref::bind_export()
- Signature: `int bind_export()`
- Purpose: Convert float anisotropy level to UI power-of-2 index
- Inputs: none (uses `m_pref`)
- Outputs/Return: int index where 2^(index-1) Γëê m_pref (0 if m_pref=0)
- Side effects: none
- Calls: none
- Notes: Treats m_pref as integer via cast; loop-based log2 computation

### AnisotropyPref::bind_import()
- Signature: `void bind_import(int value)`
- Purpose: Convert UI index to anisotropy level
- Inputs: int value (power-of-2 index)
- Outputs/Return: none (modifies `m_pref`)
- Side effects: Updates `m_pref` (sets to 2^(value-1) or 0.0 if value=0)
- Calls: none
- Notes: Result is assigned to float; value=0 ΓåÆ 0.0, value=1 ΓåÆ 1, value=2 ΓåÆ 2, etc.

### OpenGLDialog::~OpenGLDialog()
- Signature: `~OpenGLDialog()`
- Purpose: Destructor; clean up all dynamically-allocated widget pointers
- Inputs: none
- Outputs/Return: none
- Side effects: Deletes all m_*Widget members (buttons, toggles, selectors, filters)
- Calls: delete on ~20 widget pointers
- Notes: Iterates over texture and filter widget arrays; assumes all pointers are valid

### OpenGLDialog::OpenGLPrefsByRunning()
- Signature: `void OpenGLPrefsByRunning()`
- Purpose: Main entry point; orchestrate preference dialog show/run/save cycle
- Inputs: none (accesses global `graphics_preferences`)
- Outputs/Return: none (modifies global preferences and disk state)
- Side effects: Runs dialog; if accepted, persists preferences via `write_preferences()`
- Calls: m_cancelWidgetΓåÆset_callback(), m_okWidgetΓåÆset_callback(), BinderSet methods, Run(), write_preferences()
- Notes: Creates local Bindable adapters; migrate_second_to_first loads prefs into UI; migrate_first_to_second saves from UI

### SdlOpenGLDialog::SdlOpenGLDialog(int theSelectedRenderer)
- Signature: `SdlOpenGLDialog(int theSelectedRenderer)`
- Purpose: Constructor; build complete dialog UI hierarchy with all widgets, tabs, and layout
- Inputs: int theSelectedRenderer (unused in this file)
- Outputs/Return: none (initializes member widgets and dialog state)
- Side effects: Allocates ~80+ widget objects; sets up dialog placer, tabs, tables; stores widget pointers in m_*Widget members
- Calls: Allocates new instances of placers, toggles, sliders, selectors, tables; sets callbacks and labels
- Notes: Builds two tabs (General, Advanced); General tab has basic options + texture quality selectors; Advanced tab has NPOT and texture filtering controls

### SdlOpenGLDialog::Run()
- Signature: `virtual bool Run()`
- Purpose: Show dialog and run event loop
- Inputs: none
- Outputs/Return: bool (true = accepted/OK, false = cancelled)
- Side effects: Blocks until user accepts or cancels
- Calls: m_dialog.run()
- Notes: Returns true when run() returns 0 (convention for OK)

### SdlOpenGLDialog::Stop(bool result)
- Signature: `virtual void Stop(bool result)`
- Purpose: Exit dialog loop with given result
- Inputs: bool result (true = accept, false = cancel)
- Outputs/Return: none
- Side effects: Terminates dialog event loop
- Calls: m_dialog.quit()
- Notes: Passes 0 for OK (result=true), -1 for cancel (result=false)

### OpenGLDialog::Create(int theSelectedRenderer)
- Signature: `static std::unique_ptr<OpenGLDialog> Create(int theSelectedRenderer)`
- Purpose: Abstract factory; instantiate platform-specific concrete dialog
- Inputs: int theSelectedRenderer (passed to concrete constructor)
- Outputs/Return: std::unique_ptr\<OpenGLDialog\> pointing to SdlOpenGLDialog
- Side effects: Allocates new SdlOpenGLDialog on heap
- Calls: std::make_unique\<SdlOpenGLDialog\>()
- Notes: Decouples caller from concrete implementation; allows swapping UI backend at link-time

### SdlOpenGLDialog::choose_generic_tab(void *arg)
- Signature: `static void choose_generic_tab(void *arg)`
- Purpose: Callback to switch to General tab
- Inputs: void *arg (cast to SdlOpenGLDialog*)
- Outputs/Return: none
- Side effects: Changes active tab; redraws dialog
- Calls: m_tabsΓåÆchoose_tab(0), m_dialog.draw()
- Notes: Used for commented-out tab button code

### SdlOpenGLDialog::choose_advanced_tab(void *arg)
- Signature: `static void choose_advanced_tab(void *arg)`
- Purpose: Callback to switch to Advanced tab
- Inputs: void *arg (cast to SdlOpenGLDialog*)
- Outputs/Return: none
- Side effects: Changes active tab; redraws dialog
- Calls: m_tabsΓåÆchoose_tab(1), m_dialog.draw()
- Notes: Analogous to choose_generic_tab()

## Control Flow Notes
**Initialization & Frame:**
The file does not participate in main engine frame loop. Instead, it provides on-demand preference dialog functionality.

**Typical usage flow:**
1. Engine/UI calls `OpenGLDialog::Create()` to instantiate a dialog
2. Caller invokes `OpenGLPrefsByRunning()` when user opens Preferences ΓåÆ Graphics
3. Method sets up button callbacks, creates all Bindable adapter objects wrapping preference references and UI widget references, and registers them in a BinderSet
4. Calls `migrate_all_second_to_first()` to push current preference values into UI widgets
5. Calls `Run()` (virtual, overridden in SdlOpenGLDialog) to show dialog and block on user interaction
6. If user accepts, calls `migrate_all_first_to_second()` to pull UI values back to preferences, then `write_preferences()` to persist
7. On destruction, destructor cleans up all widget allocations

**Tabs & Layout:**
General tab contains toggles, sliders, and dropdowns for common settings (fog, 3D models, texture quality, anisotropy, VSync).
Advanced tab contains NPOT toggle and detailed texture filtering options (near/far filters for walls, landscapes, sprites, weapons, HUD).

## External Dependencies
- **Notable includes:**
  - `preference_dialogs.h` ΓÇö OpenGLDialog abstract base class and member variable declarations
  - `preferences.h` ΓÇö graphics_preferences_data struct and global graphics_preferences pointer
  - `binders.h` ΓÇö Bindable\<T\>, Binder\<T\>, BinderSet template classes
  - `OGL_Setup.h` ΓÇö OGL_ConfigureData, OGL_Texture_Configure, OGL_Flag_* enums, RGBColor typedef
  - `screen.h` ΓÇö Screen interface (included but not directly used in this file)
  - `<functional>` ΓÇö std::bind for callback setup
  - `<sstream>` ΓÇö std::ostringstream for formatting anisotropy display

- **External symbols (defined elsewhere):**
  - `graphics_preferences` ΓÇö global graphics_preferences_data* (read/written in OpenGLPrefsByRunning)
  - `write_preferences()` ΓÇö persists modified preferences to disk
  - `BitPref`, `BoolPref`, `Int16Pref` ΓÇö Bindable adapter classes (likely in binders.h or shared_widgets.h)
  - Widget classes: `w_toggle`, `w_select`, `w_select_popup`, `w_slider`, `w_button`, `w_label`, `w_static_text`, `w_enabling_toggle` ΓÇö UI element types
  - Dialog/layout classes: `dialog`, `vertical_placer`, `horizontal_placer`, `table_placer`, `tab_placer`, `w_tab` ΓÇö UI framework
  - Widget adapters: `ButtonWidget`, `ToggleWidget`, `SelectSelectorWidget`, `PopupSelectorWidget`, `SliderSelectorWidget` ΓÇö wrapper types (defined in shared_widgets.h)
  - Constants: `OGL_NUMBER_OF_TEXTURE_TYPES`, `OGL_Txtr_Wall`, `OGL_Txtr_Landscape`, `OGL_Txtr_Inhabitant`, `OGL_Txtr_WeaponsInHand`, `OGL_Txtr_HUD` ΓÇö texture category enums
  - `get_theme_space(ITEM_WIDGET)` ΓÇö function to query UI theme spacing
