# Penpot MCP – Architecture Review

## Overview

The Penpot MCP (Model Context Protocol) integration bridges AI language-model clients (such as Claude) with a live Penpot design project. It is structured as a TypeScript monorepo under `mcp/` containing four packages:

| Package | Location | Role |
|---------|----------|------|
| `mcp-server` | `packages/server/` | HTTP/WebSocket server exposing MCP tools to LLM clients |
| `mcp-plugin` | `packages/plugin/` | Penpot browser plugin acting as execution sandbox |
| `@penpot/mcp-common` | `packages/common/` | Shared TypeScript types for the server↔plugin protocol |
| `types-generator` | `types-generator/` | Dev-only tooling to extract Penpot API docs from source |

---

## Component Architecture

```
┌──────────────────────────────────────────────────┐
│               LLM Client (e.g. Claude)            │
│                                                  │
│  MCP protocol (Streamable HTTP or SSE)            │
└──────────────────────┬───────────────────────────┘
                       │ HTTP :4401
                       ▼
┌──────────────────────────────────────────────────┐
│            PenpotMcpServer (Express)              │
│   ┌──────────────────────────────────────────┐   │
│   │  Tools: execute_code, export_shape,       │   │
│   │         import_image, high_level_overview,│   │
│   │         penpot_api_info                   │   │
│   └──────────────┬───────────────────────────┘   │
│                  │ task dispatch                  │
│   ┌──────────────▼───────────────────────────┐   │
│   │           PluginBridge (WebSocket)        │   │
│   │      WebSocket server on :4402            │   │
│   └──────────────┬───────────────────────────┘   │
│                  │ WebSocket                      │
└──────────────────┼───────────────────────────────┘
                   │
┌──────────────────▼───────────────────────────────┐
│        Penpot MCP Plugin (browser iframe)         │
│   ┌──────────────────────────────────────────┐   │
│   │  main.ts  –  WebSocket client,            │   │
│   │             message relay                 │   │
│   └──────────────┬───────────────────────────┘   │
│                  │ postMessage                    │
│   ┌──────────────▼───────────────────────────┐   │
│   │  plugin.ts  –  task dispatch,             │   │
│   │               Penpot Plugin API calls     │   │
│   └──────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘

Additionally:
  ReplServer (Express)  :4403  – dev/debug REPL web UI
```

---

## Server (`packages/server/`)

### Entry Point (`src/index.ts`)

The entry point parses command-line arguments (only `--multi-user` is currently supported) and creates a `PenpotMcpServer` instance, then starts it. SIGINT/SIGTERM handlers call `server.stop()` for graceful shutdown.

### `PenpotMcpServer`

The central orchestrator. Key responsibilities:

* **Express HTTP server** (port `PENPOT_MCP_SERVER_PORT`, default 4401) with three endpoints:
  * `ALL /mcp` — Modern [Streamable HTTP](https://spec.modelcontextprotocol.io/specification/basic/transports/#streamable-http) MCP transport. Supports multiple concurrent sessions keyed by `mcp-session-id` header. Sessions are evicted after 60 minutes of inactivity.
  * `GET /sse` — Legacy [SSE](https://spec.modelcontextprotocol.io/specification/basic/transports/#server-sent-events-sse) MCP transport for older clients.
  * `POST /messages` — Companion POST endpoint for SSE sessions.
* **Tool registration**: creates fresh `McpServer` instances (from `@modelcontextprotocol/sdk`) per session and registers all tools.
* **Session context**: uses Node.js `AsyncLocalStorage` to propagate per-request context (user token) throughout async call chains without explicit passing.
* **Multi-user / remote / filesystem modes**: three boolean flags derived from environment variables and the `--multi-user` CLI flag control which features are available:
  * `isMultiUserMode` — requires `userToken` on both HTTP and WebSocket connections.
  * `isRemoteMode` — implies `isMultiUserMode` or `PENPOT_MCP_REMOTE_MODE=true`.
  * `isFileSystemAccessEnabled` — `true` only in local (non-remote) mode; gates `import_image` tool and file-save options in `export_shape`.

### `PluginBridge` (`src/PluginBridge.ts`)

Maintains WebSocket connections from Penpot plugin instances.

* **Connection tracking**: two indexes — `connectedClients` (socket → `ClientConnection`) and `clientsByToken` (token string → `ClientConnection`).
* **Client selection** (`getClientConnection`):
  * Single-user mode: exactly one connected client required.
  * Multi-user mode: selects by `userToken` from the current async-local session context.
* **Task dispatch** (`executePluginTask`):
  1. Serialises the task to JSON and sends it over the WebSocket.
  2. Registers the task in `pendingTasks` for response correlation.
  3. Sets a 30-second timeout.
  4. Returns a `Promise` that resolves or rejects when the plugin responds.
* **Keep-alive**: the plugin side sends a `"keep-alive"` string every 30 seconds; the bridge echoes it back.

### Tool Framework (`src/Tool.ts`)

All tools extend the abstract `Tool<TArgs>` generic base class which:

* Holds a Zod schema for input validation (schema is declared on each concrete tool's args class).
* Provides an `execute(args)` method that wraps `executeCore(args)` with logging and error handling.
* Exposes `getSessionContext()` for token-aware tools.

### Available Tools

| Tool | Class | Description |
|------|-------|-------------|
| `execute_code` | `ExecuteCodeTool` | Runs arbitrary JavaScript in the Penpot plugin sandbox; returns the value and captured console output. |
| `high_level_overview` | `HighLevelOverviewTool` | Returns the static Markdown instructions from `data/initial_instructions.md`. |
| `penpot_api_info` | `PenpotApiInfoTool` | Returns API type documentation loaded from `data/api_types.yml`. |
| `export_shape` | `ExportShapeTool` | Exports a shape as PNG or SVG; can return the image inline or save it to a file (local mode only). |
| `import_image` | `ImportImageTool` | Reads a raster image from the local file system, base64-encodes it, and injects it into the design as a Rectangle with an image fill (local mode only). |

### `ConfigurationLoader` (`src/ConfigurationLoader.ts`)

Simple file reader that loads `data/initial_instructions.md` at startup. The template variable `$api_types` is replaced at runtime with the actual list of available API type names.

### `ApiDocs` (`src/ApiDocs.ts`)

Loads `data/api_types.yml` (YAML containing Penpot API type documentation) at startup. Provides case-insensitive lookup of types and their members for use by `PenpotApiInfoTool`.

### `ReplServer` (`src/ReplServer.ts`)

A separate Express application (port `PENPOT_MCP_REPL_PORT`, default 4403) providing:

* `GET /` — serves `src/static/repl.html`, a browser-based JavaScript REPL.
* `POST /execute` — accepts `{ code: string }` JSON and executes it via `PluginBridge`, returning the result.

The REPL is intended for development and debugging; it has no authentication.

### Logging (`src/logger.ts`)

Uses [Pino](https://getpino.io/) with `pino-pretty` for both console and timestamped log-file output. The log file is created in `PENPOT_MCP_LOG_DIR` (default: `logs/`) on every server start.

---

## Plugin (`packages/plugin/`)

The plugin is a Vite-compiled project that produces two JavaScript bundles:

* **`plugin.js`** — the privileged Penpot plugin code (runs in the Penpot sandbox).
* **`index.js`** — the plugin UI, served in an invisible iframe.

### `plugin.ts` (privileged context)

* Registers exactly one task handler: `ExecuteCodeTaskHandler`.
* Opens the plugin UI in a hidden iframe.
* Relays messages between the UI and Penpot:
  * On `ui-initialized`: sends the MCP server URL and user token to the UI.
  * On `update-connection-status`: forwards the status to `mcp.setMcpStatus`.
  * On task request objects: dispatches to `handlePluginTaskRequest`.

### `main.ts` (UI iframe)

* Manages the WebSocket connection to the MCP server (`ws://…:4402`).
* Appends `?userToken=…` to the WebSocket URL in multi-user mode.
* Sends a 30-second keep-alive ping.
* Relays incoming WebSocket messages (task requests) to `plugin.ts` via `parent.postMessage`.
* Relays outgoing task responses from `plugin.ts` back to the MCP server via WebSocket.

### `ExecuteCodeTaskHandler` (`task-handlers/ExecuteCodeTaskHandler.ts`)

The sole task handler. Given a `code` string:

1. Resets the captured console log.
2. Temporarily sets `penpot.flags.naturalChildOrdering = true` (simplifies API usage).
3. Wraps the code in an async IIFE and evaluates it using `new Function()` with the execution context (`penpot`, `storage`, `console`, `penpotUtils`) injected as parameters.
4. Restores `naturalChildOrdering`, then sends the result (return value + console log) back via `task.sendSuccess(resultData)`.

### `PenpotUtils` (`src/PenpotUtils.ts`)

A static utility class injected into every code execution as `penpotUtils`. Provides helpers:

* **Shape traversal**: `findShapes`, `findShape`, `findShapeById`
* **Page helpers**: `findPage`, `getPages`, `getPageById`, `getPageByName`, `getPageForShape`
* **Shape inspection**: `shapeStructure` (deep serialisable tree), `getBounds`
* **Layout**: `addFlexLayout` (preserves child order)
* **Geometry**: `isContainedIn`, `setParentXY`
* **Styling**: `generateCss`
* **Descendant analysis**: `analyzeDescendants`
* **Image import/export**: `importImage`, `exportImage`, `base64ToByteArray`
* **Design tokens**: `findTokensByName`, `findTokenByName`, `getTokenSet`, `tokenOverview`

### `TaskHandler` / `Task` (`src/TaskHandler.ts`)

Abstract base classes for the plugin task dispatch system. `Task` encapsulates the request ID and wraps the `penpot.ui.sendMessage` response back to `main.ts`.

---

## Common Types (`packages/common/`)

Defines four TypeScript interfaces shared by the server and plugin:

* `PluginTaskRequest` — sent from server to plugin.
* `PluginTaskResponse<T>` — sent from plugin to server.
* `PluginTaskResult<T>` — resolved value of a pending task on the server side.
* `ExecuteCodeTaskParams` / `ExecuteCodeTaskResultData<T>` — typed params/result for the `executeCode` task.

---

## Data Flow: Tool Execution

```
LLM client
  │  MCP tool call: execute_code { code: "..." }
  ▼
PenpotMcpServer (Express)
  │  executeCore(args)
  ▼
PluginBridge.executePluginTask(task)
  │  JSON.stringify(task.toRequest()) → WebSocket send
  ▼
main.ts (plugin UI iframe)
  │  WebSocket message → parent.postMessage
  ▼
plugin.ts (privileged context)
  │  handlePluginTaskRequest → ExecuteCodeTaskHandler.handle
  ▼
ExecuteCodeTaskHandler
  │  new Function(…code…)()
  │  task.sendSuccess(result + log)
  ▼
penpot.ui.sendMessage({ type: "task-response", … })
  ▼
main.ts → WebSocket.send(JSON.stringify(response))
  ▼
PluginBridge.handlePluginTaskResponse
  │  task.resolveWithResult(result)
  ▼
ExecuteCodeTool.executeCore → returns TextResponse
  ▼
LLM client receives tool result
```

---

## Configuration Summary

| Variable | Default | Effect |
|----------|---------|--------|
| `PENPOT_MCP_SERVER_HOST` | `0.0.0.0` | HTTP server bind address |
| `PENPOT_MCP_SERVER_PORT` | `4401` | HTTP / SSE / Streamable HTTP port |
| `PENPOT_MCP_WEBSOCKET_PORT` | `4402` | WebSocket server port |
| `PENPOT_MCP_REPL_PORT` | `4403` | REPL server port |
| `PENPOT_MCP_REMOTE_MODE` | `false` | Disable filesystem tools |
| `PENPOT_MCP_LOG_LEVEL` | `info` | Pino log level |
| `PENPOT_MCP_LOG_DIR` | `logs` | Log file directory |

---

## Operational Modes

| Mode | Flag / Env | File System | Authentication |
|------|------------|-------------|----------------|
| Single-user (default) | — | Enabled | None (single WS client) |
| Remote | `PENPOT_MCP_REMOTE_MODE=true` | Disabled | None |
| Multi-user | `--multi-user` | Disabled | `userToken` query param |

---

## Build System

* **Package manager**: pnpm workspaces.
* **Server build**: esbuild bundles TypeScript into a single ESM file (`dist/index.js`), keeping heavy runtime dependencies (Express, ws, pino, sharp, etc.) external.
* **Plugin build**: Vite compiles and bundles TypeScript for the browser, injecting build-time constants (`IS_MULTI_USER_MODE`, `PENPOT_MCP_WEBSOCKET_URL`).
* **Top-level scripts** (`package.json`): `bootstrap` installs deps, builds all packages, and starts both servers concurrently.
