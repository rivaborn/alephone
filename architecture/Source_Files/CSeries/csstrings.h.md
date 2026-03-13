# Source_Files/CSeries/csstrings.h

## File Purpose
Header file declaring string manipulation and character encoding utility functions for the Aleph One game engine. Provides resource string loading, formatted output (printf-style), character encoding conversions (Mac Roman Γåö Unicode/UTF-8/UTF-16), and application variable expansion.

## Core Responsibilities
- Declare resource string loading from game resource sets
- Declare printf-style formatted output functions (console, debug log, file)
- Declare character encoding conversions (Mac Roman, UTF-8, UTF-16/wide)
- Declare C++ string utility wrappers
- Declare application variable substitution in strings
- Define compiler-specific printf format attributes for type safety

## Key Types / Data Structures
None (header declarations only).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `temporary` | `char[256]` | extern | Scratch buffer for temporary string operations |

## Key Functions / Methods

### csprintf
- Signature: `char *csprintf(char *buffer, const char *format, ...)`
- Purpose: Formatted string printing to a buffer (sprintf-like)
- Inputs: destination buffer, format string, variadic args
- Outputs/Return: pointer to buffer
- Side effects: writes formatted output to buffer

### dprintf
- Signature: `void dprintf(const char *format, ...)`
- Purpose: Formatted debug output (console or debug stream)
- Inputs: format string, variadic args
- Notes: Uses PRINTF_STYLE_ARGS for compiler format checking

### fdprintf
- Signature: `void fdprintf(const char *format, ...)`
- Purpose: Formatted output to file (AlephOneDebugLog.txt)
- Inputs: format string, variadic args
- Notes: Added in Aug 2001 for file-based logging

### getcstr
- Signature: `char *getcstr(char *string, short resid, size_t item)`
- Purpose: Retrieve a string from the resource set
- Inputs: destination string buffer, resource ID, item index
- Outputs/Return: pointer to destination string

### countstr
- Signature: `size_t countstr(short resid)`
- Purpose: Count items in a resource string set
- Inputs: resource ID
- Outputs/Return: count of strings in set

### Character Encoding Functions
Multiple overloaded conversion functions:
- `mac_roman_to_unicode(char)` ΓåÆ `uint16` (single char conversion)
- `mac_roman_to_unicode(const char*, uint16*)` (buffer conversion, multiple overloads with/without length limit)
- `unicode_to_mac_roman(uint16)` ΓåÆ `char` (reverse conversion)
- `mac_roman_to_utf8(const std::string&)` ΓåÆ `std::string`
- `utf8_to_mac_roman(const std::string&)` ΓåÆ `std::string`
- `utf8_to_wide(const char*|const std::string&)` ΓåÆ `std::wstring` (Windows only)
- `wide_to_utf8(const wchar_t*|const std::wstring&)` ΓåÆ `std::string` (Windows only)
- Purpose: Handle legacy Mac Roman encoding (resource compatibility) and modern UTF-8/UTF-16 for cross-platform support

### expand_app_variables
- Signature: `std::string expand_app_variables(const std::string&)` and overloads
- Purpose: Substitute application-specific variables (name, version, etc.) in strings
- Inputs: string with variable placeholders
- Outputs/Return: expanded string
- Overloads: C++ string return, in-place modification, and C-style buffer version

### build_stringvector_from_stringset
- Signature: `const std::vector<std::string> build_stringvector_from_stringset(int resid)`
- Purpose: Load all strings from a resource set into a vector
- Inputs: resource ID
- Outputs/Return: vector of strings

### copy_string_to_cstring
- Signature: `void copy_string_to_cstring(const std::string&, char*, int maxlen = 255)`
- Purpose: Convert C++ string to C-style null-terminated buffer
- Inputs: source std::string, destination buffer, max length (default 255)
- Side effects: fills destination buffer with truncated content

## Control Flow Notes
Utility header used throughout the engine wherever string operations, resource loading, or character encoding is needed. The prevalence of Mac Roman encoding suggests legacy resource format compatibility for the original Macintosh codebase. Windows-specific UTF-16 functions indicate cross-platform support was added later.

## External Dependencies
- **cstypes.h** ΓÇö defines `uint16` and other fixed-width integer types
- **SDL2/SDL_types.h** ΓÇö provides platform-neutral integer types
- **C++ standard library** ΓÇö `<string>`, `<vector>`
