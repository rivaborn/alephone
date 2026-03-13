# Source_Files/Files/FileHandler.h

## File Purpose
Provides platform-independent file, directory, and resource I/O abstractions for the Aleph One game engine. Uses SDL_RWops for underlying I/O and abstracts macOS resource fork patterns across platforms.

## Core Responsibilities
- Abstraction of file I/O operations (open, read, write, seek, close, position/length queries)
- Management of loaded resources with automatic deallocation on destruction
- File specification and path resolution (including data-directory search)
- Directory enumeration and ZIP archive listing
- Resource fork abstraction (primarily for macOS compatibility)
- File metadata queries (existence, type, modification date)
- File operations (copy, delete, rename, create directories)
- Platform-specific file dialogs (both synchronous and asynchronous)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| OpenedFile | class | Wraps SDL_RWops with position/length/error tracking |
| opened_file_device | class | Boost.IOStreams device adapter for OpenedFile |
| LoadedResource | class | Holds malloc'd resource data with RAII cleanup |
| OpenedResourceFile | class | Manages resource fork handles (macOS abstraction) |
| dir_entry | struct | Directory listing entry (name, is_directory, modification date) |
| FileSpecifier | class | Primary file path abstraction; supports creation, I/O, metadata, operations |
| ScopedSearchPath | class | RAII guard for temporary search path modification |
| DirectorySpecifier | typedef | Alias for FileSpecifier |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| unknown_filesystem_error | constexpr int | global | Sentinel value (ΓêÆ1) for unidentified filesystem errors |

## Key Functions / Methods

### OpenedFile::Read / Write
- Signature: `bool Read(int32 Count, void *Buffer);` / `bool Write(int32 Count, void *Buffer);`
- Purpose: Read/write raw bytes
- Inputs: Count (byte count), Buffer (memory location)
- Outputs/Return: Success/failure
- Side effects: Advances file position; modifies buffer (Read) or file contents (Write)
- Calls: Not inferable (SDL_RWops underneath)
- Notes: Caller responsible for buffer size ΓëÑ Count

### OpenedFile::GetPosition / SetPosition
- Signature: `bool GetPosition(int32& Position);` / `bool SetPosition(int32 Position);`
- Purpose: Query or change file position
- Inputs: Position value or reference
- Outputs/Return: Success/failure; Position output parameter filled
- Side effects: Seeks file pointer (SetPosition)
- Calls: Not inferable

### OpenedFile::GetLength / SetLength
- Signature: `bool GetLength(int32& Length);` / `bool SetLength(int32 Length);`
- Purpose: Query or modify file size
- Inputs: Length value or reference
- Outputs/Return: Success/failure; Length filled
- Side effects: May truncate file (SetLength)
- Calls: Not inferable

### OpenedFile::Close / IsOpen
- Signature: `bool Close();` / `bool IsOpen();`
- Purpose: Close file or check open state
- Inputs: None
- Outputs/Return: Success (Close); bool (IsOpen)
- Side effects: Closes handle, releases SDL_RWops
- Calls: Not inferable
- Notes: Destructor auto-closes; Close is safe to call multiple times

### FileSpecifier::SetNameWithPath
- Signature: `bool SetNameWithPath(const char *NameWithPath);`
- Purpose: Resolve and set file path, searching data directories
- Inputs: Path string (Unix-like: `dir/dir/file`, `:` ΓåÆ `/` on macOS)
- Outputs/Return: Success/failure
- Side effects: Updates internal path to first matching file in search path
- Calls: `canonicalize_path()` (private)
- Notes: Overloaded variant accepts explicit DirectorySpecifier base

### FileSpecifier::Open
- Signature: `bool Open(OpenedFile& OFile, bool Writable=false);` / `bool Open(OpenedResourceFile& OFile, bool Writable=false);` / `bool OpenForWritingText(OpenedFile& OFile);`
- Purpose: Open file or resource fork
- Inputs: OFile (output); Writable flag
- Outputs/Return: Success/failure; OFile populated
- Side effects: Opens underlying file handle via SDL_RWops
- Calls: Not inferable
- Notes: OpenForWritingText converts LFΓåÆCRLF on Windows

### FileSpecifier::ReadDialog / WriteDialog / WriteDialogAsync
- Signature: `bool ReadDialog(Typecode Type, const char *Prompt=NULL);` / `bool WriteDialog(Typecode Type, const char *Prompt=NULL, const char *DefaultName=NULL);` / `bool WriteDialogAsync(Typecode Type, char *Prompt=NULL, char *DefaultName=NULL);`
- Purpose: Display platform file selection dialogs
- Inputs: Type code; optional prompt/default name
- Outputs/Return: User confirmation result; FileSpecifier updated on success
- Side effects: UI interaction; updates internal path
- Calls: Not inferable (platform-specific)
- Notes: WriteDialogAsync allows background sound during save

### FileSpecifier::Exists / IsDir / GetDate / GetType
- Signature: `bool Exists();` / `bool IsDir();` / `TimeType GetDate();` / `Typecode GetType();`
- Purpose: Query file metadata
- Inputs: None
- Outputs/Return: bool or metadata
- Side effects: None (queries only)
- Calls: Not inferable
- Notes: GetType returns `_typecode_unknown` if unrecognized

### FileSpecifier::CopyContents / Delete / Rename / MakeDirectory
- Signature: Multiple
- Purpose: File system operations
- Inputs: Destination FileSpecifier (Copy, Rename)
- Outputs/Return: Success/failure
- Side effects: Modifies filesystem
- Calls: Not inferable

### FileSpecifier::ReadDirectory / ReadZIP
- Signature: `bool ReadDirectory(vector<dir_entry> &vec);` / `bool ReadZIP(vector<string> &vec);`
- Purpose: Enumerate directory or ZIP contents
- Inputs: Output vector reference
- Outputs/Return: Success; vector filled with entries
- Side effects: Allocates and populates vector
- Calls: Not inferable
- Notes: ReadDirectory excludes dot-prefixed files, follows symlinks; overloaded no-arg versions return vector

### FileSpecifier::SetTo*Dir
- Signature: `void SetToLocalDataDir();` (+ PreferencesDir, SavedGamesDir, QuickSavesDir, ImageCacheDir, RecordingsDir)
- Purpose: Set path to standard per-user directories
- Inputs: None
- Outputs/Return: void
- Side effects: Updates internal path
- Calls: Not inferable
- Notes: All are per-user locations

### FileSpecifier::AddPart / operator+ / operator+=
- Signature: `void AddPart(const string &part);` / `FileSpecifier operator+(const string &part) const;` / `FileSpecifier &operator+=(const string &part);`
- Purpose: Append path components
- Inputs: String, FileSpecifier, or char* path component
- Outputs/Return: void (AddPart, +=); new FileSpecifier (operator+)
- Side effects: Modifies path
- Calls: `AddPart`
- Notes: Overloaded for convenience; multiple operator variants

### LoadedResource::GetPointer / SetData / Unload / IsLoaded / GetLength
- Signature: Multiple
- Purpose: Manage malloc'd resource blocks
- Inputs: SetData takes data pointer and length
- Outputs/Return: void* (GetPointer), bool (IsLoaded), size_t (GetLength)
- Side effects: SetData assumes caller relinquishes ownership; Unload frees memory
- Calls: Not inferable
- Notes: Destructor auto-unloads; GetPointer(true) detaches ownership without deallocation

### OpenedResourceFile::Check / Get
- Signature: `bool Check(uint32 Type, int16 ID);` / `bool Get(uint32 Type, int16 ID, LoadedResource& Rsrc);`
- Purpose: Query or load a resource by type and ID
- Inputs: Type (4-char code or uint8├ù4), ID (resource ID); Rsrc (output container)
- Outputs/Return: Success/failure; Rsrc populated (Get)
- Side effects: Get allocates and loads resource
- Calls: Not inferable
- Notes: Overloaded uint8├ù4 variants via FOUR_CHARS_TO_INT macro for portability

### ScopedSearchPath (constructor / destructor)
- Signature: `ScopedSearchPath(const DirectorySpecifier& dir);` / `~ScopedSearchPath();`
- Purpose: RAII wrapper for temporary search path modification
- Inputs: Directory to insert at front of search path
- Outputs/Return: None
- Side effects: Inserts directory on construction; restores original on destruction
- Calls: Not inferable
- Notes: Copy/move constructors deleted to prevent dangling references

## Control Flow Notes

File I/O follows a standard pattern: **Open ΓåÆ Read/Write/Seek ΓåÆ Close** with RAII auto-cleanup in destructors. Resource fork access (macOS) uses **Push ΓåÆ Check/Get ΓåÆ Pop** to manage a global resource fork stack. Path resolution via `SetNameWithPath` searches the data search path, returning on first match. `ScopedSearchPath` provides lexical scoping of path modifications. Dialogs are blocking UI calls that update the FileSpecifier on user confirmation (except `WriteDialogAsync`, which allows background operation).

## External Dependencies

**Includes:**
- `tags.h` ΓÇô Typecode enum and file-type tag constants
- `<SDL2/SDL.h>` ΓÇô SDL_RWops for platform-independent I/O
- `<boost/iostreams/categories.hpp>`, `<boost/iostreams/positioning.hpp>` ΓÇô Boost.IOStreams integration
- `<vector>`, `<string>` ΓÇô STL containers
- `<time.h>` ΓÇô TimeType and time_t
- `<errno.h>` ΓÇô Error codes
- `<stddef.h>` ΓÇô size_t

**External symbols (defined elsewhere):**
- `SDL_RWops` ΓÇô SDL file abstraction struct
- `Typecode` enum, `get_typecode()`, `set_typecode()`, `FOUR_CHARS_TO_INT()` ΓÇô From tags.h
- `TimeType` ΓÇô Type alias, likely from cstypes.h
- Standard library I/O and allocation
