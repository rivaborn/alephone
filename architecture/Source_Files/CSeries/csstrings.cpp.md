# Source_Files/CSeries/csstrings.cpp

## File Purpose

String handling and localization utilities for the Aleph One game engine. Provides resource-based string retrieval with variable expansion, printf-style formatting, and character encoding conversion (Mac Roman Γåö Unicode Γåö UTF-8). Handles both legacy Mac Roman encoding and modern Unicode/UTF-8 standards.

## Core Responsibilities

- Retrieve localized strings from the TextStrings resource system by resource ID and index
- Perform printf-style string formatting
- Convert between Mac Roman (legacy Mac encoding) and Unicode/UTF-8
- Support wide character (UTF-16) conversion on Windows platforms
- Expand template variables (`$appName$`, `$appVersion$`, `$scenarioName$`, etc.) in strings
- Provide deprecated logging functions (dprintf/fdprintf) that delegate to the modern Logging framework

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `MacRomanUnicodeConverter` | class | Bidirectional converter between Mac Roman single-byte encoding and Unicode code points; uses static lookup tables and singleton pattern |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `temporary[256]` | char array | global | Buffer for temporary string storage |
| `MacRomanUnicodeConverter::mac_roman_to_unicode_table[256]` | uint16 array | static | Lookup table mapping Mac Roman byte (0ΓÇô255) to Unicode code point |
| `MacRomanUnicodeConverter::unicode_to_mac_roman` | std::map<uint16, char> | static | Reverse mapping from Unicode code points to Mac Roman bytes (0x80ΓÇô0xFF only) |
| `MacRomanUnicodeConverter::initialized` | bool | static | One-time initialization flag for populating unicode_to_mac_roman |
| `macRomanUnicodeConverter` | MacRomanUnicodeConverter | static | Singleton instance used by all conversion functions |

## Key Functions / Methods

### countstr
- Signature: `size_t countstr(short resid)`
- Purpose: Count contiguous strings in a resource set
- Inputs: `resid` ΓÇô resource ID identifying the string set
- Outputs/Return: Number of strings (starting from index 0) in that set
- Side effects: None
- Calls: `TS_CountStrings(resid)`
- Notes: Delegates to TextStrings system

### getcstr
- Signature: `char *getcstr(char *string, short resid, size_t item)`
- Purpose: Retrieve a single C string from a resource set, with variable expansion for certain resource IDs
- Inputs: `string` ΓÇô destination buffer; `resid` ΓÇô resource ID; `item` ΓÇô zero-based index within set
- Outputs/Return: Pointer to destination buffer; null-terminates result
- Side effects: Writes to destination buffer
- Calls: `TS_GetCString()`, `expand_app_variables()`
- Notes: Expands variables (`$appName$`, etc.) only for specific resource IDs (128, 129, 131, 132, 136, 138). Copies string verbatim otherwise. Returns empty string if resource not found.

### build_stringvector_from_stringset
- Signature: `const vector<string> build_stringvector_from_stringset(int resid)`
- Purpose: Load all strings from a resource set into a std::vector
- Inputs: `resid` ΓÇô resource ID
- Outputs/Return: Vector of strings (contiguous from index 0 until null is encountered)
- Side effects: Allocates vector
- Calls: `TS_GetCString()`
- Notes: Iterates from index 0 until NULL returned

### csprintf
- Signature: `char *csprintf(char *buffer, const char *format, ...)`
- Purpose: Printf-style string formatting
- Inputs: `buffer` ΓÇô destination; `format` ΓÇô format string; variadic args
- Outputs/Return: Pointer to buffer
- Side effects: Writes formatted string to buffer
- Calls: `vsprintf()`
- Notes: Unsafe; no buffer bounds checking

### dprintf / fdprintf
- Signature: `void dprintf(const char *format, ...)` / `void fdprintf(const char *format, ...)`
- Purpose: Deprecated debug logging functions; both now delegate to the Logging framework
- Inputs: Format string and variadic arguments
- Outputs/Return: None
- Side effects: Logs message via GetCurrentLogger()
- Calls: `GetCurrentLogger()->logMessageV()`
- Notes: Marked as obsolete; migration to log*() macros (Logging.h) recommended

### copy_string_to_cstring
- Signature: `void copy_string_to_cstring(const std::string &s, char *dst, int maxlen = 255)`
- Purpose: Copy std::string to C buffer with truncation
- Inputs: `s` ΓÇô source string; `dst` ΓÇô destination buffer; `maxlen` ΓÇô max characters to copy (default 255)
- Outputs/Return: None
- Side effects: Writes to dst, null-terminates
- Calls: `std::string::copy()`
- Notes: Result is automatically null-terminated

### mac_roman_to_unicode (overloads)
- Signature: 
  - `uint16 mac_roman_to_unicode(char c)`
  - `void mac_roman_to_unicode(const char *input, uint16 *output)`
  - `void mac_roman_to_unicode(const char *input, uint16 *output, int max_len)`
- Purpose: Convert Mac Roman single-byte or string to Unicode code points
- Inputs: Single byte or C string; output buffer and optional length limit
- Outputs/Return: Single code point or null-terminated code point array
- Side effects: Writes to output buffer
- Calls: `macRomanUnicodeConverter.ToUnicode()`
- Notes: String variants iterate byte-by-byte; null-terminate output array

### unicode_to_mac_roman
- Signature: `char unicode_to_mac_roman(uint16 c)`
- Purpose: Convert single Unicode code point to Mac Roman byte
- Inputs: Unicode code point
- Outputs/Return: Mac Roman byte; returns '?' if unmapped
- Side effects: None
- Calls: `macRomanUnicodeConverter.ToMacRoman()`
- Notes: Only bytes 0x80ΓÇô0xFF have reverse mappings; ASCII (0x00ΓÇô0x7F) passed through directly

### unicode_to_utf8
- Signature: `static void unicode_to_utf8(uint16 c, string& output)`
- Purpose: Encode single Unicode code point as UTF-8 bytes
- Inputs: `c` ΓÇô code point; `output` ΓÇô string to append to
- Outputs/Return: None
- Side effects: Appends 1ΓÇô3 bytes to output string
- Calls: None
- Notes: Handles code points 0x00ΓÇô0xFFFF; produces 1, 2, or 3-byte UTF-8 sequences

### utf8_to_unicode
- Signature: `static uint16 utf8_to_unicode(const char *s, int &chars_used)`
- Purpose: Decode next Unicode code point from UTF-8 sequence
- Inputs: `s` ΓÇô pointer to UTF-8 bytes; `chars_used` ΓÇô output parameter for bytes consumed
- Outputs/Return: Code point (uint16); `chars_used` set to 1, 2, or 3
- Side effects: None
- Calls: None
- Notes: Assumes well-formed UTF-8; does not validate

### mac_roman_to_utf8 / utf8_to_mac_roman
- Signatures: 
  - `std::string mac_roman_to_utf8(const std::string& input)`
  - `std::string utf8_to_mac_roman(const std::string& input)`
- Purpose: Full string conversion between Mac Roman and UTF-8
- Inputs: Source string
- Outputs/Return: Newly allocated converted string
- Side effects: Allocates string
- Calls: `mac_roman_to_unicode()`, `unicode_to_utf8()`, `utf8_to_unicode()`, `unicode_to_mac_roman()`

### utf8_to_wide / wide_to_utf8 (Windows only)
- Signatures:
  - `std::wstring utf8_to_wide(const char* utf8)` / overloads
  - `std::string wide_to_utf8(const wchar_t* utf16)` / overloads
- Purpose: Convert between UTF-8 and UTF-16 (wide character) on Windows
- Inputs: UTF-8 or UTF-16 string; length (auto-detected if not provided)
- Outputs/Return: Converted wstring or string
- Side effects: Allocates result
- Calls: Windows API (`MultiByteToWideChar`, `WideCharToMultiByte`)
- Notes: Windows only (`#ifdef __WIN32__`); uses CP_UTF8 code page

### expand_app_variables (overloads)
- Signatures:
  - `std::string expand_app_variables(const std::string& input)`
  - `void expand_app_variables(char *dest, const char *src)`
  - `void expand_app_variables_inplace(std::string& str)`
- Purpose: Substitute template variables in strings (e.g., `$appName$` ΓåÆ application name)
- Inputs: Source string; destination buffer (for C string variant)
- Outputs/Return: Expanded string; copied to dest for C variant
- Side effects: Modifies dest buffer or string in-place
- Calls: `expand_app_variables_inplace()`, `get_application_name()`, `Scenario::instance()`, `loggingFileName()`, `boost::replace_all()`
- Notes: Supports `$appName$`, `$appVersion$`, `$appLongVersion$`, `$appDate$`, `$appPlatform$`, `$appURL$`, `$appLogFile$`, `$scenarioName$`, `$scenarioVersion$`. C variant optimizes by checking for variables before conversion.

**Notes on helper/trivial functions:**
- `MacRomanUnicodeConverter::MacRomanUnicodeConverter()` ΓÇô Lazy singleton initialization; populates reverse-lookup map from forward table on first instantiation.

## Control Flow Notes

This module provides utility layer functions used throughout the engine:
- **Initialization phase**: Character encoding tables populated on first string conversion request
- **Runtime**: Called by rendering/UI systems when displaying localized strings or by logging to substitute version/scenario metadata
- **No frame/update/render cycle dependency**: Utility library, not tied to game loop

## External Dependencies

- **TextStrings.h**: `TS_GetCString()`, `TS_CountStrings()` ΓÇô resource-based string storage
- **cspaths.h**: `get_application_name()` ΓÇô application branding
- **alephversion.h**: Version constants (`A1_DISPLAY_VERSION`, `A1_VERSION_STRING`, `A1_DISPLAY_PLATFORM`, `A1_DISPLAY_DATE_VERSION`, `A1_HOMEPAGE_URL`)
- **Logging.h**: `GetCurrentLogger()`, logging levels (`logAnomalyLevel`) ΓÇô modern logging framework (used by dprintf/fdprintf)
- **Scenario.h**: `Scenario::instance()->GetName()`, `GetVersion()` ΓÇô current scenario metadata
- **boost/algorithm/string/replace.hpp**: `boost::replace_all()` ΓÇô template variable substitution
- **Windows API** (Windows only): `MultiByteToWideChar`, `WideCharToMultiByte`, `<wchar.h>`, `<windows.h>`
