# Source_Files/RenderOther/motion_sensor.h

## File Purpose
Header for the motion sensor (radar/minimap) system in the Aleph One game engine. Declares types and functions to initialize, scan, and update the motion sensor display, which shows friendly units, aliens, and enemies at different marker types.

## Core Responsibilities
- Define motion sensor display types (friend/alien/enemy classification)
- Declare motion sensor initialization with graphical assets
- Declare scanning and state management functions
- Provide change-detection for display updates
- Support runtime configuration via MML (Marathon Markup Language)
- Manage motion sensor range adjustments

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| MType_Friend | enum constant | Motion sensor marker type for friendly entities |
| MType_Alien | enum constant | Motion sensor marker type for alien entities |
| MType_Enemy | enum constant | Motion sensor marker type for hostile players |
| NUMBER_OF_MDISPTYPES | enum constant | Count of marker types |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_motion_sensor
- **Signature:** `void initialize_motion_sensor(shape_descriptor mount, shape_descriptor virgin_mounts, shape_descriptor alien, shape_descriptor _friend, shape_descriptor enemy, shape_descriptor network_compass, short side_length)`
- **Purpose:** Sets up motion sensor rendering with shape descriptors for all marker types and mount geometry.
- **Inputs:** Shape descriptors for mount geometry, three entity type markers (alien, friend, enemy), network compass sprite, side length (sensor dimensions).
- **Outputs/Return:** None.
- **Side effects:** Initializes global/static motion sensor state.
- **Calls:** Defined elsewhere (implementation in motion_sensor.c).

### reset_motion_sensor
- **Signature:** `void reset_motion_sensor(short monster_index)`
- **Purpose:** Resets sensor state for a specific monster/entity.
- **Inputs:** Monster index.
- **Outputs/Return:** None.
- **Side effects:** Clears cached state for target entity.

### motion_sensor_scan
- **Signature:** `void motion_sensor_scan(void)`
- **Purpose:** Performs a scan updateΓÇödetects entities within range and updates display state.
- **Outputs/Return:** None.
- **Side effects:** Updates internal motion sensor data.

### motion_sensor_has_changed
- **Signature:** `bool motion_sensor_has_changed(void)`
- **Purpose:** Checks if the sensor display has changed since last frame.
- **Outputs/Return:** Boolean; true if update needed.
- **Notes:** Used to optimize rendering (only redraw if changed).

### adjust_motion_sensor_range
- **Signature:** `void adjust_motion_sensor_range(void)`
- **Purpose:** Modifies motion sensor detection range (e.g., in response to upgrades or map conditions).
- **Outputs/Return:** None.
- **Side effects:** Updates sensor range parameter.

### parse_mml_motion_sensor
- **Signature:** `void parse_mml_motion_sensor(const InfoTree& root)`
- **Purpose:** Parses motion sensor configuration from an MML (Marathon Markup Language) tree.
- **Inputs:** InfoTree root node.
- **Outputs/Return:** None.
- **Side effects:** Reconfigures motion sensor from markup.

### reset_mml_motion_sensor
- **Signature:** `void reset_mml_motion_sensor()`
- **Purpose:** Resets motion sensor to default configuration (before MML parsing).
- **Outputs/Return:** None.

## Control Flow Notes
Typical frame/render cycle: `motion_sensor_scan()` runs periodically (likely per frame or per game update). `motion_sensor_has_changed()` gates redundant redraws. Configuration occurs once at startup via `initialize_motion_sensor()` and `parse_mml_motion_sensor()`.

## External Dependencies
- **shape_descriptors.h**: Provides `shape_descriptor` typedef (packed uint16 encoding collection/shape/CLUT indices).
- **InfoTree** (class): Forward-declared; defined elsewhere (likely XML/markup parsing utility).
