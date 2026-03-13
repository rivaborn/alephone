# Source_Files/Misc/Logging.h

## File Purpose
Header file for the Aleph One game engine's flexible logging facility. Defines severity levels, an abstract Logger interface, RAII context management, and convenience macros for logging messages across multiple severity tiers with automatic thread safety variants.

## Core Responsibilities
- Define eight logging severity levels (fatal through dump)
- Provide abstract Logger interface for concrete implementations
- Supply convenience macros for logging at each severity with variadic arguments
- Manage logging contexts (subcontexts) via RAII stack-based LogContext class
- Configure per-domain logging thresholds, location display, and output flushing
- Distinguish between main-thread and non-main-thread logging (NMT variants)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Logger | abstract class | Base interface for logging backends; defines push/pop context and message logging contracts |
| LogContext | class | RAII wrapper for stack-based entry/exit of logging subcontexts |
| LogAction | struct (commented) | Stub for buffered/deferred logging in SubLogger (incomplete feature) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| logDomain | extern const char* | global | Catch-all domain used by default when none specified in macros |

## Key Functions / Methods

### Logger::logMessage
- Signature: `virtual void logMessage(const char* inDomain, int inLevel, const char* inFile, int inLine, const char* inMessage, ...)`
- Purpose: Log a variadic message in main thread; delegates to virtual `logMessageV`
- Inputs: domain string, severity level enum, file/line for source location, printf-style message and args
- Outputs: None (side effect on logging backend)
- Side effects: May write to log output, flush I/O, record context stack
- Calls: `logMessageV` (virtual dispatch)

### Logger::logMessageNMT
- Signature: `virtual void logMessageNMT(const char* inDomain, int inLevel, const char* inFile, int inLine, const char* inMessage, ...)`
- Purpose: Thread-safe variant for non-main threads
- Notes: Comment notes implementation may be "more complex or missing" for thread safety

### Logger::pushLogContext
- Signature: `virtual void pushLogContext(const char* inFile, int inLine, const char* inContext, ...)`
- Purpose: Enter a named logging subcontext
- Delegates to virtual `pushLogContextV`

### LogContext::enterContextV
- Signature: `void enterContextV(const char* inFile, int inLine, const char* inContext, va_list inArgs)`
- Purpose: Push a context onto the logger's stack using variadic args
- Side effects: Calls `GetCurrentLogger()->pushLogContextV()`
- Notes: Automatically pops prior context if one is already set

### GetCurrentLogger
- Signature: `Logger* GetCurrentLogger()`
- Purpose: Retrieve the global Logger singleton
- Outputs: Non-null Logger* (defined elsewhere)

### Configuration functions
- `setLoggingThreshhold(domain, level)` ΓÇô messages appear if level < threshold
- `setShowLoggingLocations(domain, bool)` ΓÇô control file/line output
- `setFlushLoggingOutput(domain, bool)` ΓÇô flush after each message
- `parse_mml_logging(InfoTree&)`, `reset_mml_logging()` ΓÇô MML config parsing
- `loggingFileName()` ΓÇô return log file path string

## Logging Macros
**Main thread:** `logFatal`, `logError`, `logWarning`, `logAnomaly`, `logNote`, `logSummary`, `logTrace`, `logDump` ΓÇô call `logMessage` with appropriate level and `__FILE__`/`__LINE__`.

**Non-main thread:** `logErrorNMT`, `logWarningNMT`, etc. ΓÇô same but call `logMessageNMT`.

**Context management:** `logContext(...)` and `logContextNMT(...)` ΓÇô instantiate LogContext on stack with unique identifier to avoid shadowing within block scope.

## Control Flow Notes
Logging is globally accessible via `GetCurrentLogger()`. Contexts form a stack (push on entry, pop on exit via destructor). Severity levels are numeric; implementations can filter by threshold. No explicit init/shutdown visible; assumes Logger is set up before any macro invocation.

## External Dependencies
- `<stdarg.h>` ΓÇô variadic argument handling (va_list, va_start, va_end)
- `InfoTree` ΓÇô forward declared; used in MML parsing function (defined elsewhere)
- Global `GetCurrentLogger()` function ΓÇô implementation location unspecified in header
