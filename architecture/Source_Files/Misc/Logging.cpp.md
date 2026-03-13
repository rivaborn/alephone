# Source_Files/Misc/Logging.cpp

## File Purpose
Implements a flexible, thread-safe logging system for Aleph One with context stacking, severity levels, and configurable output. Provides both synchronous logging macros and context management for hierarchical debug output.

## Core Responsibilities
- Lazy initialization of the global logger instance and log file
- Stack-based context management with indentation tracking
- Thread-safe logging using per-thread context stacks and mutex protection
- Message formatting with optional file/line location information
- Severity level filtering to control log verbosity
- Optional output flushing for crash-safe logging
- Configuration via MML (Marathon Markup Language) parsing
- Dual output to both log file and stderr

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Logger` | class (abstract) | Base logging interface; defined in header |
| `TopLevelLogger` | class (concrete) | Implements Logger; manages actual logging logic |
| `LogData` | struct (nested) | Per-thread context stack and depth tracking |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sCurrentLogger` | `Logger*` | static | Global logger instance; lazily initialized |
| `sOutputFile` | `FILE*` | static | Log file handle; opened on first logger creation |
| `sLoggingThreshhold` | `int` | static | Minimum severity level to log (messages below this are squelched) |
| `sShowLocations` | `bool` | static | Flag to include source file and line numbers in output |
| `sFlushOutput` | `bool` | static | Flag to flush file after every write (defensive against crashes) |
| `logDomain` | `const char*` | global | Default logging domain ("global") |
| `g_loggingFileName` | `char[256]` | static | Cached log filename buffer |

## Key Functions / Methods

### GetCurrentLogger
- **Signature:** `Logger* GetCurrentLogger()`
- **Purpose:** Retrieve the global logger instance, initializing it on first call.
- **Inputs:** None
- **Outputs/Return:** Pointer to the global `TopLevelLogger` instance
- **Side effects:** Calls `InitializeLogging()` if `sCurrentLogger` is NULL, which opens the log file and allocates the logger
- **Calls:** `InitializeLogging()`
- **Notes:** Not thread-safe for initialization race, but safe after first call

### TopLevelLogger::pushLogContextV
- **Signature:** `void pushLogContextV(const char* inFile, int inLine, const char* inContext, va_list inArgs)`
- **Purpose:** Push a formatted context string onto the thread-local context stack.
- **Inputs:** Source file path, line number, printf-style format string, variadic arguments
- **Outputs/Return:** None
- **Side effects:** Appends context string (with optional location suffix) to per-thread `LogData::mContextStack`
- **Calls:** `getLogData()`, `vsnprintf()`, `snprintf()`
- **Notes:** Location info (file:line) is appended if `sShowLocations` is true; this is decided at push time, not log time

### TopLevelLogger::popLogContext
- **Signature:** `void popLogContext()`
- **Purpose:** Pop the most recent context from the thread-local context stack.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Removes last element from per-thread `LogData::mContextStack`; updates `mMostRecentCommonStackDepth` if stack shrinks
- **Calls:** `getLogData()`
- **Notes:** Handles shrinking stack depth to avoid re-printing contexts

### TopLevelLogger::logMessageV
- **Signature:** `void logMessageV(const char* inDomain, int inLevel, const char* inFile, int inLine, const char* inMessage, va_list inArgs)`
- **Purpose:** Format and write a log message to file and stderr, with context-based indentation.
- **Inputs:** Domain (unused), severity level, file/line info, printf-style message, variadic args
- **Outputs/Return:** None
- **Side effects:** Writes to `sOutputFile` and stderr; updates per-thread depth tracking; may flush file
- **Calls:** `getLogData()`, `vsnprintf()`, `snprintf()`, `fprintf()` (twice per context level), `fflush()`
- **Notes:** Messages are indented by `mContextStack.size() * 2` spaces; prints new context entries via "while" prefix; checks `sLoggingThreshhold` before logging; location info (file:line) appended if `sShowLocations` is true

### TopLevelLogger::flush
- **Signature:** `void flush()`
- **Purpose:** Explicitly flush the log file output.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `fflush()` on `sOutputFile` if it is open
- **Calls:** `fflush()`

### InitializeLogging
- **Signature:** `static void InitializeLogging()`
- **Purpose:** Create and initialize the global logger and open the log file.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates new `TopLevelLogger`, opens log file (creates if missing), writes timestamp header
- **Calls:** `loggingFileName()`, `fopen()` / `_wfopen()`, `time()`, `ctime()`, `fprintf()`
- **Notes:** Asserts that `sOutputFile` is NULL on entry; uses UTF-8 to wide conversion on Windows

### loggingFileName
- **Signature:** `const char *loggingFileName()`
- **Purpose:** Get or construct the log filename (cached in static buffer).
- **Inputs:** None
- **Outputs/Return:** Pointer to static buffer containing log filename
- **Side effects:** Populates `g_loggingFileName` on first call by concatenating application name + " Log.txt"
- **Calls:** `get_application_name()`, `strncpy()`, `strncat()`
- **Notes:** Returns immediately on subsequent calls; filename is assumed not to change during runtime

### setLoggingThreshhold
- **Signature:** `void setLoggingThreshhold(const char* inDomain, int16 inThreshhold)`
- **Purpose:** Configure the minimum severity level for messages to be logged.
- **Inputs:** Domain (currently ignored), threshold level
- **Outputs/Return:** None
- **Side effects:** Updates global `sLoggingThreshhold`
- **Calls:** None
- **Notes:** Domain parameter reserved for future per-domain filtering; messages at or above threshold are squelched

### setShowLoggingLocations
- **Signature:** `void setShowLoggingLocations(const char* inDomain, bool inShowLoggingLocations)`
- **Purpose:** Configure whether source file and line numbers are included in log output.
- **Inputs:** Domain (currently ignored), boolean flag
- **Outputs/Return:** None
- **Side effects:** Updates global `sShowLocations`
- **Calls:** None

### setFlushLoggingOutput
- **Signature:** `void setFlushLoggingOutput(const char* inDomain, bool inFlushOutput)`
- **Purpose:** Configure whether output file is flushed after each write.
- **Inputs:** Domain (currently ignored), boolean flag
- **Outputs/Return:** None
- **Side effects:** Updates global `sFlushOutput`; calls `fflush()` immediately if enabling flush mode
- **Calls:** `fflush()`

### parse_mml_logging
- **Signature:** `void parse_mml_logging(const InfoTree& root)`
- **Purpose:** Parse MML (Marathon Markup Language) logging configuration from XML structure.
- **Inputs:** InfoTree node containing `<logging_domain>` elements
- **Outputs/Return:** None
- **Side effects:** For each `<logging_domain>` child, reads domain name and optional attributes; calls setter functions to apply configuration
- **Calls:** `setLoggingThreshhold()`, `setShowLoggingLocations()`, `setFlushLoggingOutput()`
- **Notes:** Skips entries with empty or missing domain attribute; supports optional `threshhold`, `show_locations`, and `flush` attributes

## Control Flow Notes
**Initialization:** Logger creation is lazyΓÇöthe first call to any logging macro triggers `GetCurrentLogger()`, which calls `InitializeLogging()` exactly once.

**Per-Thread Flow:** Each thread maintains its own `LogData` via `std::unordered_map` keyed by thread ID, protected by a static mutex in `getLogData()`. Context push/pop and message logging operate on thread-local stacks.

**Logging:** When a message is logged, new context levels (since the last log) are printed with "while" prefixes, then the message is printed indented by context depth. File and line info are appended if enabled.

**Configuration:** Logging behavior (threshold, location display, flushing) can be set via C++ API calls or parsed from MML at runtime.

## External Dependencies
- **Standard library:** `<thread>`, `<mutex>`, `<unordered_map>`, `<vector>`, `<string>`, `<fstream>`, `<time.h>`, `<stdio.h>`, `<stdarg.h>`
- **Aleph One engine:** `Logging.h` (header), `cseries.h`, `shell.h`, `FileHandler.h`, `InfoTree.h` (for MML parsing)
- **Platform-specific:** `<unistd.h>`, `<sys/types.h>`, `<pwd.h>` (Unix/Mac/BSD); Windows-specific `_wfopen()` and UTF-8 conversion utilities (defined elsewhere)
- **External symbols used:** `log_dir` (DirectorySpecifier), `get_application_name()` (string), `utf8_to_wide()` (converter)
