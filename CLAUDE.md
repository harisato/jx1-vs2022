# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **Sword Online (Vo Lam 1 / 武林Online)** — a Vietnamese martial arts MMORPG originally by Kingsoft, ported to Visual Studio 2022. The codebase is C++ targeting Win32 (x86). Screen resolution has been expanded to 1600x900.

## Build

**Solution file:** `SwordOnline/Sources/JXAll.sln` (Visual Studio 2022)

Four build configurations (all Win32 only):
- **Debug** / **Release** — builds server components (Bishop, GameServer, Heaven, Goddess, Rainbow, S3Relay, Common, etc.)
- **Client Debug** / **Client Release** — builds client components (S3Client, Represent3, Core, Engine, etc.)

All output binaries go to the `/bin` directory.

**MSBuild from command line:**
```
msbuild SwordOnline/Sources/JXAll.sln /p:Configuration="Debug" /p:Platform="Win32"
msbuild SwordOnline/Sources/JXAll.sln /p:Configuration="Client Debug" /p:Platform="Win32"
```

**Dependencies:** DirectX 9 SDK (bundled in `dx9dsdk/`), precompiled libraries in `SwordOnline/Lib/`. No package manager — all dependencies are vendored.

## Architecture

The system is a distributed microservice architecture where Bishop launches and orchestrates the other server processes.

### Server components (`SwordOnline/Sources/MultiServer/`)

| Component | Type | Role |
|-----------|------|------|
| **Bishop** | GUI app | Authentication server, master orchestrator. Launches Heaven/Goddess/Rainbow as DLLs. Configure credentials in `Bishop/Application.cpp` line 78 (default: "fsjx"/"1234"). |
| **GameServer** | Console app | Main game logic server. Handles player sessions, NPCs, combat, scripting. Connects to Bishop on startup. |
| **Heaven** | DLL | Gateway server — connection routing hub between all servers. |
| **Goddess** | DLL | Database server — persists player/game data (MSSQL via ODBC/ADO). |
| **Rainbow** | DLL | Relay/chat server — inter-server messaging and player chat. |
| **S3Relay** | Console app | Distributed relay server for cross-server communication. |
| **Common** | Static lib | Shared networking (Winsock2/IOCP), threading, buffer management, encryption. Used by all server components. |

### Client (`SwordOnline/Sources/S3Client/`)
- Game client using DirectX 9 rendering via the `Represent3` DLL
- UI system in `S3Client/Ui/`
- Network connection via `S3Client/Src/LoginDef.h` and connection agent

### Shared libraries
- **Core** (`Sources/Core/`) — Static lib with game data definitions, player mechanics, item systems, NPC logic, Lua scripting integration. Shared between client and server (conditional compilation via build config).
- **Engine** (`Sources/Engine/`) — Static lib with file I/O, cryptography, pak file handling, sprite/image loading.
- **Represent3** (`Sources/Represent/Represent3/`) — DLL for 3D rendering (DirectX 9).
- **LuaLibDll** (`Sources/Library/LuaLib/`) — Lua scripting engine.

### Payment system (`Sources/Sword3PaySys/`)
- **Sword3PaySys** (S3AccServer) — Account/payment server
- **S3RelayServer** — Payment relay

## Key Files

- **Protocol definitions:** `SwordOnline/Headers/KProtocol.h` (~53KB, 207+ message types) — the central protocol file that defines all client-server and inter-server communication structures. Also `KProtocolDef.h`, `KTongProtocol.h` (guilds), `KGmProtocol.h` (GM commands), `KRelayProtocol.h`.
- **Server/client interfaces:** `SwordOnline/Headers/IServer.h`, `IClient.h` — abstract networking interfaces.
- **Game constants:** `SwordOnline/Sources/Core/Src/GameDataDef.h` — max players (1200/server), max NPCs (48000 server / 256 client), max items (160000 server / 512 client), FPS (18), team size (7), guild limits, region/cell sizes.
- **Server config:** `SwordOnline/Sources/MultiServer/GameServer/ServerCfg.ini`

## Data flow

1. **Player login:** Client → Bishop (auth) → Heaven (gateway) → GameServer (session)
2. **Inter-server:** GameServer ↔ Heaven ↔ Rainbow (chat/relay) ↔ other GameServers
3. **Persistence:** GameServer → Heaven → Goddess (DB read/write)
4. **Cross-server:** S3Relay handles relay between multiple server clusters

## Code Style

Uses `.clang-format` with Microsoft style:
- 4-space indentation, no tabs
- Allman brace style
- 400 character column limit
- Include order is intentionally preserved (`SortIncludes: false`) — do not reorder includes
- No short blocks/functions/ifs on single lines

## Important Notes

- All source is C++ (Win32 API). No cross-platform support.
- The codebase uses Hungarian notation throughout (e.g., `m_nPlayerCount`, `g_pServer`, `BOOL bResult`).
- Protocol structs use `#pragma pack` for binary serialization — changing struct layouts breaks network compatibility.
- Core library compiles differently for client vs server via preprocessor defines in the respective build configurations.
- Comments and some documentation are in Vietnamese.
- No automated test suite exists in this codebase.
