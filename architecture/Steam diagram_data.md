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

## External Dependencies
- **Windows API**: CreateProcessW, CreatePipe, WriteFile, ReadFile, SetEnvironmentVariableA, CloseHandle, ExitProcess, MessageBoxA
- **POSIX**: fork, execvp, pipe, read, write, waitpid, signal, _exit
- **Boost**: boost::filesystem, boost::dll::program_location, boost::regex
- **Steamworks SDK**: steam_api.h (SteamAPI_InitEx, SteamAPI_RunCallbacks, SteamUserStats/UGC/User/Utils/Friends/Apps, callback types)
- **std**: string, sstream, vector


