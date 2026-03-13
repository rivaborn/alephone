# Source_Files/XML/XML_ParseTreeRoot.h

## File Purpose
Declares the root element of the XML parse tree for the Aleph One game engine and provides the public interface for parsing Marathon Markup Language (MML) from files and in-memory buffers. This is the entry point for MML configuration loading during engine initialization.

## Core Responsibilities
- Export the public API for MML parsing (`ParseMMLFromFile`, `ParseMMLFromData`)
- Declare the configuration reset function (`ResetAllMMLValues`)
- Define the absolute root element containing all valid XML document trees
- Support both file-based and buffer-based MML parsing

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### ResetAllMMLValues
- Signature: `extern void ResetAllMMLValues();`
- Purpose: Reset all MML-modified settings back to hard-coded engine defaults
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies global/static state throughout the engine to revert MML-applied configuration changes
- Calls: Not inferable from this file
- Notes: Typically called before parsing a new MML file or when resetting to defaults

### ParseMMLFromFile
- Signature: `extern bool ParseMMLFromFile(const FileSpecifier& filespec, bool load_menu_mml_only);`
- Purpose: Parse MML configuration from a file on disk
- Inputs: `filespec` (file reference), `load_menu_mml_only` (flag to parse only menu-related MML)
- Outputs/Return: `bool` indicating parse success/failure
- Side effects: Modifies global configuration state based on parsed MML directives
- Calls: Not inferable from this file
- Notes: File format validation and error handling delegated to implementation

### ParseMMLFromData
- Signature: `extern bool ParseMMLFromData(const char *buffer, size_t buflen);`
- Purpose: Parse MML configuration from an in-memory buffer
- Inputs: `buffer` (pointer to data), `buflen` (buffer length in bytes)
- Outputs/Return: `bool` indicating parse success/failure
- Side effects: Modifies global configuration state based on parsed MML directives
- Calls: Not inferable from this file
- Notes: Allows MML parsing from embedded/packaged data or network sources

## Control Flow Notes
Part of the engine's initialization phase. MML is typically parsed early during startup to configure game parameters. The `load_menu_mml_only` flag suggests conditional parsing during different startup stages (menu vs. in-game).

## External Dependencies
- `FileSpecifier` (forward declaration; defined elsewhere in engine)
- Standard library: `<stddef.h>` (for `size_t`)
- Comment references GPL 3.0 licensing
