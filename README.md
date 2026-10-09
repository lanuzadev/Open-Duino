# Open Duino

**The AI-native Arduino IDE.** Open Duino is a full Arduino IDE 2.x with a
Model Context Protocol (MCP) server built into its core — so an AI agent like
OpenCode can write sketches, compile them, upload to real boards, read the serial
monitor, and manage libraries *alongside you*, editing the same files you see in
the editor in real time.

There's no plugin to install into the AI, no sidecar process, no copy-pasting
code back and forth. **The IDE itself is the agent's workbench.** You launch it,
point your assistant at `http://127.0.0.1:3847`, and the two of you share one
editor, one board, one serial monitor.

<sub>Built on [Arduino IDE 2.x](https://github.com/arduino/arduino-ide) · An
independent community project, not affiliated with or endorsed by Arduino SA ·
AGPL-3.0</sub>

---

## Why it exists

The Arduino IDE is where embedded projects get built. Modern AI agents are great
at embedded code — but they work blind, guessing at your board, your errors, and
your wiring, and handing you snippets to paste. Open Duino closes that gap by
making the IDE a first-class participant in the conversation:

- **The agent sees what you see** — the open sketch, the selected board and port,
  connected devices, and real compiler output (not a guess).
- **You see what the agent does** — when it writes a file, the editor opens it,
  reloads it, and shows a *"Created by Claude"* notification. True pair
  programming on hardware.
- **It drives the real toolchain** — the same `arduino-cli`, clang-format, and
  serial monitor the IDE uses. Compile results, memory usage, and upload status
  are the genuine article.

## Download

Open Duino is being developed from this source checkout. Packaged downloads
will be published after the first Open Duino release. Until then, follow
[Building from source](#building-from-source).

> **Got the official Arduino IDE too?** They run side by side as separate apps.
> They share what should be shared, your sketchbook, installed boards and
> libraries, and preferences (`~/.arduinoIDE`), but each keeps its own window
> state and opens independently.
>
> The upstream Arduino IDE and Open Duino can run side by side. They share the
> sketchbook, installed boards, libraries, and preferences in `~/.arduinoIDE`.

## Quick start — connect your agent

1. **Launch Open Duino.** The MCP server starts automatically on
   `http://127.0.0.1:3847`. The status bar (bottom left) shows **MCP ready**,
   and **Agent active** once an assistant is using it.

2. **Help › Connect an AI Agent…** (or click that status bar item). It gives
   you copy-ready setups with your installation's real paths already filled in:
   - **OpenCode**: one `opencode mcp add` command to paste into a terminal,
     plus a configuration entry for manual setup.
   - **Claude Code**: one `claude mcp add` command to paste into a terminal.
   - **Claude Desktop, Cursor and other MCP clients**: an `mcpServers` entry
     for the client's config file.
   - **Direct HTTP**: for clients that speak Streamable HTTP, with your token.

   The OpenCode and Claude Code commands use the bundled stdio bridge, so the client stays
   connected while Open Duino is closed and starts it when a tool needs it
   (needs Node.js 18 or newer).

   <details>
   <summary>Prefer to write the config yourself?</summary>

   Via the bridge (recommended), on Windows:

   ```json
   {
     "mcpServers": {
       "arduino": {
         "command": "node",
         "args": ["C:/path/to/Open Duino/resources/mcp-bridge/bridge/arduino-agent-bridge.js"],
         "env": { "ARDUINO_AGENT_PATH": "C:/path/to/Open Duino/Open Duino.exe" }
       }
     }
   }
   ```

   On macOS the bridge is at
   `Open Duino.app/Contents/Resources/mcp-bridge/bridge/arduino-agent-bridge.js`;
   on Linux, under `resources/` in the unzipped folder. From a source checkout,
   use `arduino-mcp-extension/bridge/arduino-agent-bridge.js` after building the
   extension. The bridge reads the token from `~/.arduinoIDE/mcp-token` itself
   and has no dependencies; see [the bridge README](arduino-mcp-extension/bridge/README.md)
   for its options.

   OpenCode's generated local config uses `mcp.servers.open-duino` with a
   `node` command pointing at the bundled bridge. Use the generated command or
   config in **Help › Connect an AI Agent…**; Node.js 18 or newer is required.

   Direct HTTP (only connects while Open Duino is running):

   ```json
   {
     "mcpServers": {
       "arduino": {
         "type": "http",
         "url": "http://127.0.0.1:3847/mcp",
         "headers": { "Authorization": "Bearer <token from ~/.arduinoIDE/mcp-token>" }
       }
     }
   }
   ```

   </details>

   > **Claude Code users:** the server sends workflow guidance automatically,
   > and ships three slash commands (`/bringup`, `/debug-serial`,
   > `/profile-board`). For deeper hardware know-how, install the bundled
   > skill: copy [`skills/arduino-agent/`](skills/arduino-agent/) into
   > `~/.claude/skills/`.

3. **Open the in-app agent.** Choose **Help › OpenCode Chat** to open OpenCode's
   TUI in Open Duino's integrated terminal, alongside the sketch editor. It uses
   the current sketch folder as its workspace, and edits appear in the editor
   as they are written. Connect the bridge in step 2 first and configure an
   OpenCode provider if you have not already.

4. **Talk to your board.**
   - *"Create a Blink sketch and open it."*
   - *"What boards are connected?"*
   - *"Compile for the Uno and explain any errors."*
   - *"Upload it, then show me the serial output at 115200."*

## What the agent can do

| Category | Operations |
|----------|------------|
| **Sketches** | Create, open, and edit sketches; read/write code; browse and clone built-in examples |
| **Build** | Compile with `wait:true` for one-call results; live progress; real compiler output with structured, explained errors |
| **Upload** | Compile + flash in one call; failures come back explained (bootloader mode, busy port, wrong FQBN, power) |
| **Boards** | Detect connected boards with USB vid/pid; identify unknown boards (`suggest_fqbn`); pin capabilities; install cores |
| **Serial** | Cursor-based lossless reads; `wait_for` a pattern; automatic crash/reset/watchdog/brownout detection |
| **Libraries** | Search the registry; install/remove; browse library examples |
| **Formatting** | Format Arduino/C++ with clang-format |
| **Config** | Sketchbook location, board-manager URLs, IDE settings |

By default the tools are exposed through a **router pattern** — 4 meta-tools
(`list_tool_categories`, `get_category_tools`, `execute_tool`, `search_tools`)
so the agent discovers tools on demand instead of loading every definition into
its context. A **direct mode** exposes all tools individually if you prefer.

## How it works

```
+------------------+     HTTP (Bearer auth)     +---------------------------+
|    AI agent      | <------------------------> |       Open Duino          |
|  (MCP client)    |   http://127.0.0.1:3847    |  (Theia/Electron + MCP)    |
+------------------+           /mcp             +------------------------------+
                                                      |
                                             Arduino toolchain
                                        (arduino-cli daemon, clang-format,
                                         pluggable serial monitor)
```

The MCP server is embedded in the IDE's backend and speaks the modern
**Streamable HTTP** transport (plus legacy SSE for older clients), supporting
multiple simultaneous sessions.

**Security is on by default:**
- Binds to `127.0.0.1` only.
- Requires a bearer token (generated on first launch, stored in
  `~/.arduinoIDE/mcp-token`).
- Rejects browser-originated requests and sends no CORS headers, so a web page
  can't reach it.
- Confines file access to your sketchbook and the built-in examples.

The only unauthenticated endpoint is a health check:

```bash
curl http://127.0.0.1:3847/health
```

## Settings

**File → Preferences → MCP:**

| Setting | Description | Default |
|---------|-------------|---------|
| Enable MCP server | Turn the integration on/off | `true` |
| Start automatically | Launch the server with the IDE | `true` |
| Server port | HTTP port (1024–65535) | `3847` |
| Require auth | Require the bearer token | `true` |
| Log level | none / error / info / debug | `info` |
| Tool mode | Router (4 meta-tools) or Direct (all tools) | `router` |

## Made for learning, too

Open Duino ships extras aimed at STEM and classroom use:

- **Example browser** — every built-in Arduino example, with descriptions.
- **Hardware reference** — ask for a board's pin map, PWM/I2C/SPI pins, memory.
- **Beginner-friendly errors** — compiler errors returned with plain-language
  explanations and suggested fixes.

It also carries a modernized UI (refined buttons, dialogs, board selector,
progress bars, and serial monitor) that respects both light and dark themes.

## Building from source

**Prerequisites:** Node.js 18+, Yarn 4 (via Corepack), Python 3.11, Go 1.21, and
a C/C++ toolchain (VS 2022 Build Tools on Windows).

```bash
git clone https://github.com/lanuzadev/Open-Duino.git
cd Open-Duino

corepack enable
yarn install
yarn prepare:shims      # create the launcher shims Theia's build expects

yarn build:dev          # build all packages, including the MCP extension
cd electron-app && yarn start
```

Windows has a few extra native-module notes (mostly automated now) — see
[**docs/BUILDING-WINDOWS.md**](docs/BUILDING-WINDOWS.md). Full extension
documentation lives in
[**arduino-mcp-extension/README.md**](arduino-mcp-extension/README.md).

## Project status

Open Duino inherits an MCP workflow previously verified against real hardware
by its upstream project — an ESP32-S3 (N16R8) driven end to end through the MCP
tools alone. This fork's branded build has not yet been packaged or revalidated:

> install the board core → create a sketch → write the code → compile →
> upload → read the board's own serial output

Every step ran as an MCP call, with no manual work in the IDE. In one pass the
agent wrote a WiFi scanner, flashed it, and read 21 access points back off the
board.

Those sessions shaped the tooling itself. v0.6.0 turned every pain point they
surfaced into a feature: `wait:true` replaces polling loops, serial reads are
lossless and cursor-based, crashes/resets/watchdogs are detected and reported
as events, upload failures come back explained, and `suggest_fqbn` identifies
boards arduino-cli can't. The server also now teaches connected agents its own
workflow (instructions at connect, `/bringup`-style prompts, a Claude Code
skill).

The release workflow is configured to build **Windows, macOS and Linux**
archives for Open Duino tags. Packaged downloads will be published after the
first release.

**Known limits:** artifacts are unsigned. Boards whose USB VID/PID appear in no
`boards.txt` (many ESP32-S3 devkits) can't be auto-identified — use
`arduino_board suggest_fqbn`, then pass the FQBN explicitly. Uploading over a
board's *native* USB port can require manual bootloader entry; a UART bridge
port works without it.

Contributions and bug reports are welcome through this repository's issue
tracker and pull requests.

## Relationship to the Arduino IDE

Open Duino is a fork of the open-source
**[Arduino IDE 2.x](https://github.com/arduino/arduino-ide)** (a
[Theia](https://theia-ide.org/)/[Electron](https://www.electronjs.org/)
application that drives the [arduino-cli](https://github.com/arduino/arduino-cli)).
All of the core IDE work is theirs; this project adds the embedded MCP server,
the AI-collaboration features, and the UI refinements on top.

**Arduino® is a trademark of Arduino SA.** Open Duino is an independent,
community project and is **not affiliated with, sponsored by, or endorsed by
Arduino SA.** The name describes this project's purpose — an agent-driven Arduino
development environment — and implies no official connection.

## License

Licensed under the **GNU AGPL-3.0-or-later**, the same license as the upstream
Arduino IDE. Distributions include third-party components under compatible
licenses (GPLv2, MIT, BSD-3). See [LICENSE.txt](LICENSE.txt).
