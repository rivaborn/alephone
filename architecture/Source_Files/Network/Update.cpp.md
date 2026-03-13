# Source_Files/Network/Update.cpp

## File Purpose
Implements the Update class, which checks for new software versions by fetching update metadata over HTTP in a background thread. It compares the remote version with the local version and reports whether an update is available.

## Core Responsibilities
- Manage initialization and lifecycle of the update-check background thread
- Perform HTTP GET requests to a remote update server
- Parse version metadata (date version and display version) from HTTP responses
- Compare semantic versions to detect available updates
- Provide thread-safe access to update status and version information via singleton pattern

## Key Types / Data Structures
None defined in this file. Uses `Status` enum (defined in header) and manages string/thread member variables.

## Global / File-Static State
None. State is managed by the `Update` singleton instance (see `Update::instance()` in header).

## Key Functions / Methods

### Update (constructor)
- **Signature:** `Update() : m_status(NoUpdateAvailable), m_thread(0)`
- **Purpose:** Initialize the singleton instance and trigger the first update check.
- **Side effects:** Calls `StartUpdateCheck()`; spawns a background thread.

### ~Update (destructor)
- **Signature:** `~Update()`
- **Purpose:** Clean shutdown; wait for and terminate the background update thread.
- **Side effects:** Calls `SDL_WaitThread()` on `m_thread` if it exists.

### StartUpdateCheck
- **Signature:** `void StartUpdateCheck()`
- **Purpose:** Initiate a new update check. If one is already in progress, returns early.
- **Inputs:** None (uses implicit member state).
- **Outputs/Return:** None.
- **Side effects:** Sets `m_status` to `CheckingForUpdate`; clears `m_new_date_version` and `m_new_display_version`; creates new SDL thread.
- **Calls:** `SDL_WaitThread()`, `SDL_CreateThread()`.
- **Notes:** Waits for any prior thread before creating a new one. Returns early if status is already `CheckingForUpdate`.

### update_thread (static)
- **Signature:** `static int update_thread(void *p)`
- **Purpose:** Static thread entry point; casts void pointer and delegates to instance method.
- **Inputs:** `p` ΓÇô pointer to `Update` instance.
- **Outputs/Return:** Return value from `Thread()`.
- **Calls:** `Thread()` (via cast).
- **Notes:** Required by SDL's thread creation API.

### Thread
- **Signature:** `int Thread()`
- **Purpose:** Worker thread function that fetches and parses version data from the remote server.
- **Inputs:** None (uses implicit member state and constants).
- **Outputs/Return:** `0` on success, `1` on HTTP failure, `5` on missing date version.
- **Side effects:** Sets `m_status` to `UpdateAvailable`, `NoUpdateAvailable`, or `UpdateCheckFailed`; populates `m_new_date_version` and `m_new_display_version`.
- **Calls:** `HTTPClient::Get()`, `HTTPClient::Response()`, `boost::tokenizer` parsing.
- **Notes:** Parses response line-by-line; expects lines prefixed with `"A1_DATE_VERSION: "` and `"A1_DISPLAY_VERSION: "`; compares date versions lexicographically.

## Control Flow Notes
This is initialization-phase code operating asynchronously. The constructor triggers `StartUpdateCheck()`, which spawns a background thread running `Thread()`. The main thread can poll `GetStatus()` to determine when the check completes. All state transitions route through the `m_status` member.

## External Dependencies
- **HTTPClient** (`HTTP.h`) ΓÇô Makes GET requests and buffers the response.
- **SDL2/SDL_thread.h** ΓÇô Thread creation and synchronization (`SDL_CreateThread`, `SDL_WaitThread`).
- **boost::tokenizer, boost::algorithm::string::predicate** ΓÇô String parsing (tokenization by newlines, prefix matching).
- **alephversion.h** ΓÇô Provides `A1_UPDATE_URL` and `A1_DATE_VERSION` constants.
