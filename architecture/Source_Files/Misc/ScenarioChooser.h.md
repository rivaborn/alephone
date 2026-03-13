# Source_Files/Misc/ScenarioChooser.h

## File Purpose
Defines the `ScenarioChooser` class, which manages a scrollable grid UI for displaying and selecting game scenarios. It handles layout calculations, user input, and rendering via SDL2.

## Core Responsibilities
- Aggregate scenarios from multiple sources (primary, workshop, directories)
- Calculate and manage grid layout (rows, columns, scrolling offset)
- Handle keyboard/input navigation and selection
- Render scenario previews in a grid with proper spacing
- Return the user's selected scenario and its metadata

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ScenarioChooserScenario` | class (forward decl.) | Encapsulates a single scenario's metadata and preview image |
| `font_info` | class (forward decl.) | Font resource for text rendering |

## Global / File-Static State
None.

## Key Functions / Methods

### ScenarioChooser (constructor)
- **Signature:** `ScenarioChooser()`
- **Purpose:** Initialize chooser with default dimensions and empty scenario list
- **Side effects:** Allocates data structures; does not open a window

### run()
- **Signature:** `std::pair<std::string, bool> run()`
- **Purpose:** Main event loop; displays chooser UI and waits for user selection
- **Outputs/Return:** Path to selected scenario and workshop flag
- **Side effects:** Creates SDL window, handles events, redraws UI in loop
- **Notes:** Likely blocks until user makes a selection or quits

### add_primary_scenario / add_workshop_scenario / add_directory
- **Purpose:** Populate scenario list from paths
- **Inputs:** File path(s)
- **Side effects:** Appends to `scenarios_` vector; may trigger layout recalculation

### num_scenarios()
- **Purpose:** Query scenario count
- **Outputs/Return:** Integer count

## Control Flow Notes
The class is designed as a modal UI component: construction initializes state, `run()` blocks to present the chooser, and the caller uses the returned scenario path. Layout recalculation and rendering occur during `run()` in response to user input (arrow keys likely) and window resizing.

## External Dependencies
- `<SDL2/SDL.h>` ΓÇô windowing, events, rendering
- `<string>`, `<vector>`, `<tuple>` ΓÇô standard library containers
- Forward declarations: `ScenarioChooserScenario`, `font_info` (defined elsewhere)
