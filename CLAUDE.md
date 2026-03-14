# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Windows-MCP is a Python MCP (Model Context Protocol) server that bridges AI LLM agents with the Windows OS, enabling direct desktop automation. It exposes 18 tools (App, PowerShell, FileSystem, Snapshot, Click, Type, Scroll, Move, Shortcut, Wait, Scrape, MultiSelect, MultiEdit, Clipboard, Process, Notification, Registry, LocateText) via FastMCP.

## Build & Development Commands

```bash
uv sync                    # Install dependencies
uv run windows-mcp         # Run the MCP server
ruff format .              # Format code
ruff check .               # Lint code
ruff check --fix .         # Lint and auto-fix
pytest                     # Run all tests (if tests/ exists)
pytest tests/test_foo.py   # Run a single test file
```

**Package manager**: UV (not pip). **Python**: 3.13+. **Build backend**: Hatchling.

## Architecture

The codebase follows a layered service architecture under `src/windows_mcp/`:

**Entry point** — `__main__.py`: Registers all 18 MCP tools on a FastMCP server instance. Uses an async lifespan to initialize Desktop, WatchDog, and Analytics services. Each tool function delegates to `Desktop` methods. The `@with_analytics` decorator wraps tools for telemetry. Supports two modes via `MODE` env var: `local` (default, direct desktop access) and `remote` (proxies to a dashboard endpoint using `API_KEY` + `SANDBOX_ID`). CLI flags: `--transport` (stdio/sse/streamable-http), `--host`, `--port`.

**Desktop service** — `desktop/service.py`: High-level orchestrator. Manages window operations (launch, resize, switch), screenshots, mouse/keyboard actions, and clipboard. Interfaces with Tree service for UI element discovery. `desktop/views.py` defines data models: `DesktopState`, `Window`, `Size`, `BoundingBox`, `Status`.

**Tree service** — `tree/service.py`: Captures the Windows accessibility tree from active and background windows. Identifies interactive elements and scrollable areas. Uses `ThreadPoolExecutor` for multi-threaded UI traversal. `tree/views.py` defines `TreeElementNode`, `ScrollElementNode`, `TreeState`. `tree/config.py` has control type configurations.

**UIAutomation wrapper** — `uia/`: Low-level abstraction over the Windows UIAutomation COM API via `comtypes`. `core.py` wraps the main automation object, `controls.py` has control-specific logic, `patterns.py` wraps UIAutomation patterns, `enums.py` has COM enumerations, `events.py` handles event subscriptions.

**WatchDog** — `watchdog/service.py`: Runs in a separate thread monitoring UI focus changes via UIAutomation events. Notifies the Tree service of focus changes so the accessibility tree stays current.

**Virtual Desktop Manager** — `vdm/core.py`: Tracks which windows belong to which Windows virtual desktop (Win10/11).

**Filesystem service** — `filesystem/service.py`: Structured file operations (read, write, copy, move, delete, list, search, info) exposed via the `FileSystem` tool. Relative paths are resolved against the user's Desktop folder. Max read size is enforced via `MAX_READ_SIZE` in `filesystem/views.py`.

**Auth service** — `auth/service.py`: `AuthClient` handles remote-mode authentication. POSTs `api_key` + `sandbox_id` to the dashboard, receives a `session_token`, then `ProxyClient` uses it as a Bearer token to forward MCP calls to the dashboard's `/api/mcp` endpoint. Retries with exponential backoff on transient failures; fails fast on 4xx errors.

**Analytics** — `analytics.py`: Optional PostHog telemetry (disabled with `ANONYMIZED_TELEMETRY=false` env var). Tracks tool names and errors only, not arguments or outputs.

## Code Style

- Formatter/linter: **Ruff** (line length 100, double quotes)
- Naming: PEP 8 — `snake_case` functions/variables, `PascalCase` classes, `UPPER_CASE` constants
- Type hints required on function signatures
- Google-style docstrings for public functions/classes

## Key Design Details

- Screenshots are capped to 1920x1080 for token efficiency
- Mouse/keyboard input uses UIA (same coordinate space as BoundingRectangle; no DPI mismatch)
- Browser detection (Chrome, Edge, Firefox) triggers special DOM extraction mode in Snapshot
- Fuzzy string matching (`thefuzz`) is used for element name matching
- UI element fetching has retry logic (`THREAD_MAX_RETRIES=3` in tree service)
- The server supports stdio, SSE, and streamable HTTP transports
- `bool` tool parameters also accept string `"true"`/`"false"` (agents may send booleans as strings)
- The `Registry` tool reads/writes the Windows Registry via PowerShell; paths use PowerShell format (e.g. `HKCU:\Software\MyApp`)
- Remote mode proxies all MCP calls through a dashboard; the local desktop tools are not used in that mode

## Security Context

This server has **full system access** with no sandboxing. Tools like Shell and App can perform irreversible operations. The recommended deployment target is a VM or Windows Sandbox.
