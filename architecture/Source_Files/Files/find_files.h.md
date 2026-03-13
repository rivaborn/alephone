# Source_Files/Files/find_files.h

## File Purpose
Defines an abstract file-finder framework and a concrete implementation for collecting files from the filesystem. Uses a template-method pattern where `FileFinder` conducts recursive directory traversal and invokes a virtual callback for each file found.

## Core Responsibilities
- Abstract base class `FileFinder` that defines the contract for file search operations with recursive directory support
- Virtual callback `found()` that subclasses override to handle discovered files
- Concrete implementation `FindAllFiles` that appends all discovered files to a provided `std::vector`
- Support for wildcard file type matching via `WILDCARD_TYPE` constant

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileFinder` | class | Abstract base class; defines file search interface with recursive traversal |
| `FindAllFiles` | class | Concrete subclass that accumulates found files into a vector |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `WILDCARD_TYPE` | `Typecode` | global | Constant for matching any file type during search (mapped to `_typecode_unknown`) |

## Key Functions / Methods

### FileFinder::Find
- **Signature:** `bool Find(DirectorySpecifier &dir, Typecode type, bool recursive = true)`
- **Purpose:** Initiates a recursive (or non-recursive) file search in a specified directory for files of a given type.
- **Inputs:** `dir` (target directory), `type` (file type filter; use `WILDCARD_TYPE` for all), `recursive` (default true)
- **Outputs/Return:** `bool` (success of search initiation)
- **Side effects:** Iterates through directory contents and invokes virtual `found()` callback for each matching file.
- **Notes:** Search can be aborted early if `found()` returns true.

### FileFinder::found (pure virtual)
- **Signature:** `virtual bool found(FileSpecifier &file) = 0`
- **Purpose:** Callback invoked for each file discovered during search; must be overridden by subclasses.
- **Inputs:** `file` (FileSpecifier reference to discovered file)
- **Outputs/Return:** `bool` (true aborts search; false continues)

### FindAllFiles::found
- **Signature:** `bool found(FileSpecifier &file)` (override)
- **Purpose:** Appends the discovered file to the internal destination vector.
- **Inputs:** `file` (FileSpecifier reference)
- **Outputs/Return:** `bool` (always false to continue searching)
- **Side effects:** Appends to `dest_vector`; constructor clears the vector on instantiation.

## Control Flow Notes
- Client creates a `FindAllFiles` instance with a reference to a target `std::vector<FileSpecifier>`
- Client calls `Find()` with a directory, type filter, and recursion flag
- For each file found, `found()` is invoked; results accumulate in the vector
- Designed for integration into game asset discovery / resource-loading pipelines

## External Dependencies
- **FileHandler.h** ΓÇô provides `FileSpecifier`, `DirectorySpecifier`, `Typecode` constants, and file I/O abstractions
- **tags.h** (via FileHandler.h) ΓÇô defines typecode constants (e.g., `_typecode_unknown`)
- **\<vector\>** ΓÇô standard library container for file results
