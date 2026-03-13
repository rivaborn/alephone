# Source_Files/XML/InfoTree.cpp

## File Purpose
Implements serialization and deserialization for game configuration data using Boost.PropertyTree, supporting XML and INI formats. Provides conversion utilities between file representations and game-specific types (colors, shapes, damage, fonts, paths, angles).

## Core Responsibilities
- Load/save XML and INI configuration files from `FileSpecifier` paths or streams
- Convert between game-specific numeric types and tree representation (fixed-point, world units, angles)
- Encode/decode paths (symbolic path expansion/contraction) and strings (Mac Roman Γåö UTF-8)
- Serialize/deserialize structured game data (colors, shapes, damage definitions, fonts)
- Provide iteration over named child nodes with range adaptor

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `InfoTreeFileStream` | class | RAII wrapper around `opened_file_device` for managed XML/INI file I/O |
| `RGBColor` | struct (from cseries.h) | 16-bit RGB color with red, green, blue components |
| `rgb_color` | struct (from cseries.h) | Alternative RGB color type |
| `shape_descriptor` | typedef (from world.h) | Packed descriptor for collection, CLUT, and sequence |
| `damage_definition` | struct (from world.h) | Damage type, flags, base amount, random, scale |
| `FontSpecifier` | struct (from FontHandler.h) | Font size, style, file path |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ColorToTree` | `float` | static | Conversion factor (1/65535) for normalizing uint16 colors to [0,1] range |
| `TreeToColor` | `float` | static | Conversion factor (65535) for denormalizing [0,1] floats to uint16 |

## Key Functions / Methods

### load_xml (static, file variant)
- Signature: `static InfoTree load_xml(FileSpecifier filename)`
- Purpose: Load and parse XML configuration file into tree structure
- Inputs: `FileSpecifier` file path
- Outputs/Return: New `InfoTree` populated from XML
- Side effects: File I/O; throws `InfoTree::unexpected_error` on open failure
- Calls: `InfoTreeFileStream()`, `pt::read_xml<pt::iptree>()`
- Notes: Uses Boost.PropertyTree XML parser; case-insensitive element matching

### load_xml (static, stream variant)
- Signature: `static InfoTree load_xml(std::istringstream& stream)`
- Purpose: Parse XML from in-memory string stream
- Inputs: Reference to input string stream
- Outputs/Return: New `InfoTree` populated from stream
- Side effects: Reads from stream; throws on malformed XML
- Calls: `pt::read_xml<pt::iptree>()`

### save_xml / save_ini (instance methods)
- Signature: `void save_xml(FileSpecifier filename) const`; `void save_ini(FileSpecifier filename) const`
- Purpose: Serialize tree to XML/INI file with indentation (XML only)
- Inputs: File path; `const` reference implies tree is not modified
- Outputs/Return: Void; side effect is file written
- Side effects: File I/O; throws on write failure; `write_indented_xml` uses 2-space indent
- Calls: `InfoTreeFileStream()`, `write_indented_xml()` / `pt::write_ini()`
- Notes: Boost version 1.56+ uses `std::string` for XML settings; earlier versions use `char`

### read_fixed / read_wu / read_angle
- Signature: `bool read_fixed(string path, _fixed& value, float min, float max) const`
- Purpose: Read bounded floating-point value and convert to fixed-point (1/1) or world unit (1/1024) representation
- Inputs: Attribute path, reference to output variable, bounds
- Outputs/Return: `true` if attribute found and in range; output written; `false` otherwise
- Side effects: Modifies output parameter; angle normalizes to [0, 360) degrees
- Calls: `read_attr_bounded()`
- Notes: `read_angle` applies modulo to force range; multiplies by `FULL_CIRCLE` constant

### read_path (2 overloads)
- Signature: `bool read_path(string key, FileSpecifier& file) const`; `bool read_path(string key, char *dest) const`
- Purpose: Read file path from attribute, optionally expanding symbolic paths (e.g., `$APP`)
- Inputs: Attribute key, output buffer/FileSpecifier
- Outputs/Return: `true` if attribute found; `false` otherwise
- Side effects: Modifies output; calls `expand_symbolic_paths()` for C-string variant (max 255 chars)
- Calls: `read_attr()`, `SetNameWithPath()` or `expand_symbolic_paths()`

### read_cstr / put_cstr / put_attr_cstr
- Signature: `bool read_cstr(string key, char *dest, int maxlen) const`; `void put_cstr(string key, string cstr)`
- Purpose: Read/write C strings with Mac Roman Γåö UTF-8 encoding conversion
- Inputs: Key, destination buffer (read) or source string (write); max length for read
- Outputs/Return: `true` if attribute found (read); void (write)
- Side effects: Encoding conversion via `DeUTF8_C()` and `mac_roman_to_utf8()`
- Calls: `read_attr()`, `put_attr()` or `put()`

### read_color / add_color (overloads)
- Signature: `bool read_color(RGBColor& color) const`; `void add_color(string path, const RGBColor& color [, size_t index])`
- Purpose: Read RGB color from three separate red/green/blue attributes; add color as child node
- Inputs: Reference to color struct; optional index for indexed colors
- Outputs/Return: `true` if any color component found (read); void (write)
- Side effects: Modifies output on read; creates child node on write
- Calls: `_get_color()`, `add_child()`, `_make_color()`
- Notes: Components stored as floats in [0,1] range; converted to/from uint16; template instantiated for both `RGBColor` and `rgb_color`

### read_shape
- Signature: `bool read_shape(shape_descriptor& descriptor, bool allow_empty = true) const`
- Purpose: Read shape/collection/CLUT descriptor from attributes; validate against max indices
- Inputs: Reference to descriptor; flag to permit empty (UNONE) descriptor
- Outputs/Return: `true` if valid descriptor assembled; `false` if invalid/incomplete
- Side effects: Modifies output; reads three separate attributes (`seq`/`frame`, `coll`, `clut`)
- Calls: `read_attr_bounded()`, `BUILD_DESCRIPTOR()`, `BUILD_COLLECTION()`
- Notes: Allows `seq` or `frame` attribute name (both point to sequence index); clut defaults to 0 if absent

### read_damage / read_font
- Signature: `bool read_damage(damage_definition& def) const`; `bool read_font(FontSpecifier& font) const`
- Purpose: Read structured game data from multiple child nodes/attributes
- Inputs: Reference to output struct
- Outputs/Return: `true` if any field found; accumulates status across multiple reads
- Side effects: Modifies output fields; calls `Update()` on font after reading
- Calls: `read_indexed()`, `read_attr()`, `read_fixed()`

### children_named
- Signature: `const_child_range children_named(string key) const`
- Purpose: Return iterator range over all child nodes with a given key name
- Inputs: Element name
- Outputs/Return: `const_child_range` (forwarding range of `const InfoTree` objects)
- Side effects: None (const)
- Calls: `equal_range()`, `boost::make_iterator_range()`, `boost::adaptors::values()`
- Notes: Uses case-insensitive matching; returns forward-traversal range

## Control Flow Notes
Utility layer; no frame/update/render cycle. Invoked during engine initialization and configuration reloads. `InfoTreeFileStream` constructor throws exceptions on file errors, preventing silent corruption. Template methods in header dispatch to overloaded implementations based on type traits.

## External Dependencies
- **Boost.PropertyTree** (`<boost/property_tree/*.hpp>`) ΓÇô core tree structure and XML/INI parsers
- **Boost.Iostreams** (`<boost/iostreams/stream.hpp>`) ΓÇô stream abstraction for file I/O
- **Boost.Range** (`<boost/range/adaptor/map.hpp>`) ΓÇô range adaptor for map-like iteration
- **Game engine utilities** ΓÇô `FileHandler.h` (`FileSpecifier`, `opened_file_device`), `FontHandler.h` (`FontSpecifier`), `TextStrings.h` (`DeUTF8_C`, `mac_roman_to_utf8`), `shell.h` (`expand_symbolic_paths`, `contract_symbolic_paths`)
- **Game data types** (defined elsewhere) ΓÇô `_fixed`, `angle`, `shape_descriptor`, `damage_definition`, `RGBColor`, `rgb_color`
