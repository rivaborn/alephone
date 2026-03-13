# Source_Files/XML/InfoTree.h

## File Purpose

InfoTree wraps Boost's PropertyTree (iptree) to provide a type-safe, exception-safe API for reading and writing game configuration data from XML and INI files. It specializes in parsing Aleph One game world types (colors, shapes, damage definitions, fonts, angles) from structured data, serving as a bridge between file I/O and the game engine's data model.

## Core Responsibilities

- Load/save XML and INI configuration files via Boost PropertyTree
- Provide type-safe template readers with fallback defaults for arbitrary types
- Parse game-specific types: colors, shape descriptors, damage definitions, fonts, fixed-point angles
- Read XML attributes with bounds checking and enum validation
- Handle attribute paths with proper XML namespace prefixes (`<xmlattr>.`)
- Write attributes and game-specific data back to tree structures
- Expose named child element iteration via `children_named()`

## Key Types / Data Structures

| Name | Kind (struct/enum/class/typedef/interface/trait) | Purpose |
|------|---|---|
| InfoTree | class | Inherits boost::property_tree::iptree; main API for structured data I/O |
| parse_error | typedef | XML parser exception type |
| ini_error | typedef | INI parser exception type |
| path_error | typedef | PropertyTree path lookup exception |
| data_error | typedef | PropertyTree type conversion exception |
| unexpected_error | typedef | Generic PropertyTree exception |
| const_child_range | typedef | Boost range over const InfoTree children |

## Global / File-Static State

None.

## Key Functions / Methods

### load_xml (static)
- Signature: `static InfoTree load_xml(FileSpecifier filename)` and `static InfoTree load_xml(std::istringstream& stream)`
- Purpose: Parse XML file or stream into InfoTree structure
- Inputs: File path or input stream
- Outputs/Return: New InfoTree instance populated with parsed data
- Side effects: File I/O, may throw parse_error
- Calls: Boost XML parser

### save_xml
- Signature: `void save_xml(FileSpecifier filename) const` and `void save_xml(std::ostringstream& stream) const`
- Purpose: Serialize InfoTree to XML file or stream
- Inputs: Output destination (file or stream)
- Outputs/Return: None
- Side effects: File I/O or stream modification
- Calls: Boost XML writer

### read (template)
- Signature: `template<typename T> bool read(std::string path, T& value) const`
- Purpose: Type-safe generic reader; silently returns false on path or conversion errors
- Inputs: Property path string, reference to output variable
- Outputs/Return: true if successful, false if path not found or type conversion failed
- Side effects: Modifies output parameter only on success
- Calls: get_child(), get_value<T>()
- Notes: Preferred over direct PropertyTree access; requires template instantiation for each type

### read_attr / read_attr_bounded (templates)
- Signature: `template<typename T> bool read_attr(std::string path, T& value)` and overloads for bounded versions
- Purpose: Read XML attribute with optional bounds validation; separate overload for enums
- Inputs: Attribute path, output reference, optional min/max bounds
- Outputs/Return: true if in bounds, false otherwise
- Side effects: Modifies output only on success
- Calls: read() with `<xmlattr>.` prefix prepended
- Notes: Enum version reads as underlying_type and casts to enum; respects numeric bounds

### read_indexed
- Signature: `bool read_indexed(std::string path, int16& value, int num_slots, bool allow_none = false) const`
- Purpose: Read and validate array index within [0, num_slots-1] or [NONE, num_slots-1]
- Inputs: Path, num_slots, allow_none flag
- Outputs/Return: true if index in valid range
- Calls: read_attr_bounded() with computed bounds

### Specialized readers (read_color, read_shape, read_damage, read_font, read_path, read_fixed, read_wu, read_angle)
- Purpose: Parse game-specific types from property tree
- Inputs: Path string; some take output references, some default parameters (min/max for numeric types)
- Outputs/Return: bool indicating success; output by reference
- Calls: Delegates to underlying game type parsers (not visible in this header)
- Notes: Must be defined in implementation file; provide domain-specific error handling

### put_attr (template)
- Signature: `template<typename T> void put_attr(std::string path, const T value)`
- Purpose: Write arbitrary type as XML attribute via PropertyTree
- Inputs: Attribute path, value to serialize
- Outputs/Return: None
- Calls: put() with `<xmlattr>.` prefix
- Notes: Inverse of read_attr()

### add_color, put_attr_path, put_cstr, put_attr_cstr
- Purpose: Specialized writers for colors, file paths, C-strings
- Inputs: Path and value(s); add_color has indexed overloads
- Outputs/Return: None (void)
- Calls: put() or internal serialization functions
- Notes: add_color supports both RGBColor and rgb_color types; indexed variants for arrays

### children_named (const)
- Signature: `const_child_range children_named(std::string key) const`
- Purpose: Iterate over all child elements with a given tag name
- Inputs: Child element name
- Outputs/Return: Boost forward range of const InfoTree references
- Side effects: None (lazy; iteration may raise path_error)
- Calls: Boost range utilities
- Notes: Returns lightweight range object; enables range-based for loops

## Control Flow Notes

InfoTree is instantiated during level initialization and map loading (inferred from dependencies on `map.h`, `world.h`). It is used to parse XML/INI configuration files that define game entities, properties, and parameters. Load operations occur at startup or level transition; save operations may occur when writing modified configurations or saving game state. The class serves as an adapter between Boost's generic PropertyTree library and Aleph One's game-specific data types, isolating type conversion and error handling logic.

## External Dependencies

- `<boost/property_tree/ptree.hpp>`, `<boost/property_tree/ini_parser.hpp>`, `<boost/property_tree/xml_parser.hpp>` ΓÇö XML/INI parsing and serialization
- `<boost/range/any_range.hpp>` ΓÇö Range adaptation for child iteration
- `<type_traits>` ΓÇö SFINAE overloading for enum vs. non-enum read_attr_bounded
- `FileHandler.h` ΓÇö FileSpecifier for abstract file paths
- `FontHandler.h` ΓÇö FontSpecifier type for font configuration
- `map.h`, `world.h` ΓÇö Game world types (shape_descriptor, damage_definition, angle, _fixed, RGBColor)
- `cseries.h` ΓÇö Base types and macros (RGBColor struct, NONE constant)
