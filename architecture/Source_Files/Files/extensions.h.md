# Source_Files/Files/extensions.h

## File Purpose
Header interface for physics file management in the Aleph One game engine (Marathon engine port). Provides functions to load physics definitions from files, handle network physics data, and compute file checksums for validation.

## Core Responsibilities
- Set and configure physics data files for the engine
- Import and process physics definition structures from files
- Manage network-synchronized physics buffers
- Validate physics data integrity via checksums
- Support version control of physics data formats

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | class | Encapsulates file path/reference for physics data files (forward-declared; defined elsewhere) |

## Global / File-Static State
None.

## Key Functions / Methods

### set_physics_file
- **Signature:** `void set_physics_file(FileSpecifier& File)`
- **Purpose:** Configure which physics data file the engine should read from.
- **Inputs:** File specifier reference pointing to a physics data file.
- **Outputs/Return:** None.
- **Side effects:** Updates internal engine state to use the specified file for subsequent physics operations.
- **Calls:** Not visible in header.
- **Notes:** Takes a reference; caller retains ownership. Likely called at engine startup or in response to mod/configuration changes.

### set_to_default_physics_file
- **Signature:** `void set_to_default_physics_file(void)`
- **Purpose:** Reset physics file to the engine's default/built-in physics data.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Resets internal physics file state.
- **Calls:** Not visible in header.
- **Notes:** No-argument convenience function, likely used for initialization or reset scenarios.

### import_definition_structures
- **Signature:** `void import_definition_structures(void)`
- **Purpose:** Parse and load all physics definition structures from the currently-set physics file.
- **Inputs:** None (uses internal state set by `set_physics_file`).
- **Outputs/Return:** None (modifies engine state in-place).
- **Side effects:** Populates engine physics definitions (monsters, weapons, projectiles, etc.); may allocate memory.
- **Calls:** Not visible in header.
- **Notes:** Called after setting a physics file, likely during init or mod reload.

### get_network_physics_buffer
- **Signature:** `void *get_network_physics_buffer(int32 *physics_length)`
- **Purpose:** Retrieve a serialized buffer of physics data for network transmission.
- **Inputs:** Pointer to `int32` for output length.
- **Outputs/Return:** Opaque void pointer to buffer; caller receives its byte length via output parameter.
- **Side effects:** Allocates or returns pointer to internal physics buffer.
- **Calls:** Not visible in header.
- **Notes:** Used in multiplayer to sync physics state. Caller must know buffer lifetime/ownership.

### process_network_physics_model
- **Signature:** `void process_network_physics_model(void *data)`
- **Purpose:** Deserialize and apply physics data received from network.
- **Inputs:** Opaque buffer containing serialized physics model.
- **Outputs/Return:** None.
- **Side effects:** Updates engine physics definitions; may trigger model rebuilding or validation.
- **Calls:** Not visible in header.
- **Notes:** Inverse of `get_network_physics_buffer`; used in multiplayer to receive peer physics state.

### get_physics_file_checksum
- **Signature:** `uint32_t get_physics_file_checksum()`
- **Purpose:** Compute a checksum of the currently-loaded physics file for validation/versioning.
- **Inputs:** None.
- **Outputs/Return:** 32-bit unsigned checksum value.
- **Side effects:** None.
- **Calls:** Not visible in header.
- **Notes:** Likely used to validate consistency between network peers or detect physics file modifications.

## Control Flow Notes
Physics file loading is a startup/initialization step:
1. `set_physics_file()` or `set_to_default_physics_file()` configures the source.
2. `import_definition_structures()` parses and populates engine state.
3. In multiplayer, `get_network_physics_buffer()` sends local state; `process_network_physics_model()` receives peer state.
4. `get_physics_file_checksum()` validates consistency.

Not directly tied to per-frame update/render; primarily initialization and inter-process synchronization.

## External Dependencies
- **cstypes.h** ΓÇö provides `int32`, `uint32_t` fixed-size integer types via SDL_types.h
- **FileSpecifier** ΓÇö forward-declared class; implementation defined elsewhere (likely in file manager / OS abstraction layer)

---

**Version Constants:**
- `BUNGIE_PHYSICS_DATA_VERSION = 0` (legacy)
- `PHYSICS_DATA_VERSION = 1` (current)
