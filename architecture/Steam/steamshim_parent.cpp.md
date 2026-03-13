# Steam/steamshim_parent.cpp

## File Purpose
Parent process that acts as a Steam API bridge for a game. Initializes the Steam SDK, spawns a child game process, and communicates with it via pipes to relay Steam API calls, achievements, stats, and workshop operations.

## Core Responsibilities
- Platform-specific process spawning and bidirectional pipe I/O (Windows and Unix)
- Steam API initialization and management of interface pointers
- Binary protocol command parsing and dispatching from child process
- Steam async callback handling and result relay back to child
- Workshop item lifecycle (upload, update, query, delete)
- Achievement and statistics operations persistence
- Game overlay activation monitoring
- Environment variable setup for child process integration

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| ProcessType | typedef | Platform-abstracted process handle |
| PipeType | typedef | Platform-abstracted pipe descriptor |
| steam_game_information | struct | Game installation path and workshop support flag |
| item_subscribed_query_result | struct | Query result for subscribed workshop items with installation paths |
| item_owned_query_result | struct | Query result for user's published workshop items |
| item_upload_data | struct | Workshop item metadata for creation/update |
| ItemType | enum class | Workshop item category (Scenario, Plugin, Map, Physics, Script, Sounds, Shapes) |
| ContentType | enum class | Item content classification (Graphics, HUD, Music, etc.) |
| SteamBridge | class | Manages Steam callback registration and async result handling |
| ShimCmd / ShimEvent | enum | Binary protocol command and event type identifiers |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| GSteamStats | ISteamUserStats* | static | Steam user stats interface |
| GSteamUGC | ISteamUGC* | static | Steam workshop interface |
| GSteamUser | ISteamUser* | static | Steam user identity interface |
| GSteamFriends | ISteamFriends* | static | Steam friends/overlay interface |
| GSteamUtils | ISteamUtils* | static | Steam utilities interface |
| GSteamBridge | SteamBridge* | static | Callback bridge instance |
| GAppID | AppId_t | static | Marathon Infinity app ID |
| GUserID | uint64 | static | Current Steam user ID |
| game_info | steam_game_information | static | Installation folder and feature flags |
| GArgc, GArgv | int/char** | static | Unix command-line state |
| GlpCmdLine | LPWSTR | static | Windows command-line state |

## Key Functions / Methods

### mainline
- Signature: `static int mainline(void)`
- Purpose: Orchestrate initialization, child launch, and shutdown
- Inputs: None (uses global state)
- Outputs/Return: Child process exit code
- Side effects: Creates pipes, initializes Steam, spawns child, enters command loop, cleans up
- Calls: `createPipes()`, `initSteamworks()`, `launchChild()`, `setEnvironmentVars()`, `processCommands()`, `deinitSteamworks()`, `closeProcess()`, `fail()`

### initSteamworks
- Signature: `static bool initSteamworks(PipeType fd, ESteamAPIInitResult*, SteamErrMsg*)`
- Purpose: Initialize Steam SDK and cache interface pointers
- Side effects: Calls `SteamAPI_InitEx()`, caches global interface pointers, creates SteamBridge, queries app ID and user ID
- Calls: `SteamAPI_InitEx()`, `SteamUserStats()`, `SteamUtils()`, `SteamUser()`, `SteamUGC()`, `SteamFriends()`, `SteamApps()->GetAppInstallDir()`

### launchChild
- Signature: `static bool launchChild(ProcessType *pid)`
- Purpose: Spawn child game process
- Inputs: pid output parameter
- Side effects: Calls `fork()` / `CreateProcessW()`, child execs game binary (Classic Marathon.exe or alephone)
- Calls: `findExe()`, `fork()` + `execvp()` or `CreateProcessW()`
- Notes: Uses regex to locate executable; child calls `_exit(1)` on exec failure

### processCommands
- Signature: `static void processCommands(PipeType pipeParentRead, PipeType pipeParentWrite)`
- Purpose: Main blocking loop reading commands from child
- Side effects: Reads from pipe, dispatches commands, maintains framing buffer
- Calls: `readPipe()`, `processCommand()`
- Notes: Implements length-prefixed framing (2-byte little-endian length + payload)

### processCommand
- Signature: `static bool processCommand(const uint8 *buf, unsigned int buflen, PipeType fd)`
- Purpose: Dispatch command to Steam API and return result
- Outputs/Return: false to terminate parent, true to continue
- Side effects: Modifies global stat/achievement state; writes response to pipe
- Calls: `SteamAPI_RunCallbacks()`, `SteamStats->SetAchievement/ClearAchievement()`, `SteamStats->SetStat/GetStat()`, `GSteamUGC->CreateItem()`, workshop query functions, various `writeXxx()` functions
- Notes: Many comments flag buffer overflow risks; no length validation on string fields

### SteamBridge callback methods
- `OnUserStatsStored()`, `OnOverlayActivated()`, `OnItemCreated()`, `OnItemUpdated()`, `OnItemOwnedQueried()`, `OnItemModQueried()`, `OnItemScenarioQueried()`
- Purpose: Async callback handlers; serialize results and write to pipe
- Side effects: Invoke `writeXxx()` functions, populate internal result sets for pagination
- Calls: `write*()` functions, `GSteamUGC->ReleaseQueryUGCRequest()`, workshop query continuations

### SteamBridge::idle
- Purpose: Poll and report workshop upload progress
- Calls: `GSteamUGC->GetItemUpdateProgress()`, `write3ByteCmd()`

### Workshop helpers
- `UpdateItem()`, `workshopQueryItemOwned()`, `workshopQueryItemMod()`, `workshopQueryItemScenario()`, `GetTagsForItemType()`
- Purpose: Construct and submit UGC operations to Steam
- Calls: `GSteamUGC` methods, `GSteamBridge->set_item_*_callback()` to register async handlers
- Notes: Pagination support (resubmits next page if `kNumUGCResultsPerPage` results returned)

### Platform abstraction helpers
- `writePipe()`, `readPipe()`, `createPipes()`, `closePipe()`, `setEnvVar()`, `findExe()`
- Purpose: Encapsulate Windows and Unix system calls
- Notes: Read/write on Unix retry on EINTR; Windows handles inherited pipes for child

## Control Flow Notes
**Startup**: Entry ΓåÆ `mainline()` ΓåÆ Steam init ΓåÆ pipes ΓåÆ fork/CreateProcess ΓåÆ child exec  
**Runtime**: Parent blocks in `processCommands()` ΓåÆ read frame ΓåÆ `processCommand()` ΓåÆ Steam call ΓåÆ write result  
Steam callbacks fire asynchronously during `SteamAPI_RunCallbacks()` (called on SHIMCMD_PUMP)  
**Shutdown**: Child closes pipe ΓåÆ loop exits ΓåÆ shutdown Steam ΓåÆ wait child ΓåÆ return exit code

## External Dependencies
- **Windows API**: CreateProcessW, CreatePipe, WriteFile, ReadFile, SetEnvironmentVariableA, CloseHandle, ExitProcess, MessageBoxA
- **POSIX**: fork, execvp, pipe, read, write, waitpid, signal, _exit
- **Boost**: boost::filesystem, boost::dll::program_location, boost::regex
- **Steamworks SDK**: steam_api.h (SteamAPI_InitEx, SteamAPI_RunCallbacks, SteamUserStats/UGC/User/Utils/Friends/Apps, callback types)
- **std**: string, sstream, vector
