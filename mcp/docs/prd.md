# Penpot MCP – Product Requirements Document (PRD)

**Status**: In Development  
**Version**: 1.0  
**Last Updated**: 2026-03-10

---

## 1. Problem Statement

Design and development workflows increasingly rely on AI assistants to draft, iterate, and inspect artefacts. Penpot is an open-source, browser-based design tool, but today's AI assistants cannot directly interact with a live Penpot project: they can only generate descriptions or static code snippets that require human copy-paste to apply.

The Penpot MCP server closes this gap by exposing a machine-readable interface – built on the [Model Context Protocol (MCP)](https://spec.modelcontextprotocol.io/) – that allows any MCP-compatible AI client to query, create, and modify Penpot designs in real time.

---

## 2. Goals

1. **AI-driven design creation** – An LLM can create and modify shapes, apply fills, set layout, and organise layers without human copy-paste.
2. **AI-driven design inspection** – An LLM can read the current design structure, export images, and retrieve CSS to answer questions about the design or feed that information into code generation.
3. **Safety by default** – Filesystem access and multi-user functionality are disabled by default; they must be explicitly opted into.
4. **Extensibility** – New capabilities can be added by implementing the `Tool` interface; no changes to the protocol layer are needed.
5. **Low friction** – A single `pnpm run bootstrap` command installs dependencies, builds all components, and starts all servers.

---

## 3. Non-Goals

* Penpot MCP does **not** replace or duplicate the Penpot web application UI.
* Penpot MCP does **not** provide user account management or authentication beyond the simple `userToken` mechanism in multi-user mode.
* Penpot MCP does **not** implement real-time collaboration conflict resolution.
* Penpot MCP does **not** support vector-drawing input from the AI (e.g. SVG path authoring); shapes are created via the Penpot Plugin API.

---

## 4. User Personas

| Persona | Description | Primary Need |
|---------|-------------|--------------|
| **Designer** | Uses Penpot daily; wants AI help for repetitive tasks (duplicating components, applying bulk style changes, generating boilerplate layouts). | Trigger AI actions from their AI chat tool without leaving Penpot. |
| **Developer** | Works alongside designers; wants to ask an AI to extract CSS, check spacing, or scaffold React component stubs from designs. | Design inspection + code generation in one workflow. |
| **AI Platform Operator** | Deploys the MCP server centrally for a team. | Secure multi-user mode; no per-user server instance. |

---

## 5. Features

### 5.1 Core Tool: `execute_code`

**Description**: The primary power-user tool. Accepts a JavaScript code string and executes it inside the Penpot Plugin sandbox using `new Function()`.

**Input**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `code` | `string` | Yes | JavaScript code to execute. Has access to `penpot` (Penpot Plugin API), `penpotUtils` (utility helpers), and `storage` (cross-call persistent object). |

**Output**: The `return` value of the code (JSON-serialised) and captured `console` output.

**Context variables available to the executed code**:

| Variable | Type | Description |
|----------|------|-------------|
| `penpot` | `Penpot` | Full Penpot Plugin API object. |
| `penpotUtils` | `PenpotUtils` | Static utility class with shape-traversal, image, layout, and token helpers. |
| `storage` | `object` | Persistent object preserved across tool calls in the same session (can store any JS value, including functions). |
| `console` | `ExecuteCodeTaskConsole` | Custom console implementation; all output is captured and returned alongside the result. |

**Behaviour**:
* `naturalChildOrdering` flag is set to `true` during execution for simplified API usage.
* Exceptions are caught; the error message is returned instead of a result.

---

### 5.2 Tool: `high_level_overview`

**Description**: Returns the Markdown instruction document loaded from `data/initial_instructions.md`. This document teaches the LLM how to use the Penpot Plugin API correctly (shape properties, hierarchy, layout, z-order, fills, etc.).

**Input**: None.

**Output**: Markdown text containing the high-level instructions.

**Usage guidance**: The LLM is instructed to call this tool at least once before using `execute_code`, so it understands the API conventions.

---

### 5.3 Tool: `penpot_api_info`

**Description**: Returns structured API documentation for a specific Penpot Plugin API type or type member, loaded from `data/api_types.yml`.

**Input**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `type` | `string` | Yes | Name of the API type to look up (case-insensitive). |
| `member` | `string` | No | Optional specific member (property or method) name. |

**Output**:
* If `member` is given: documentation for that specific member.
* Otherwise: full type documentation (if ≤ 2000 chars) or an overview with a prompt to call again with a `member` name.

---

### 5.4 Tool: `export_shape`

**Description**: Exports a Penpot shape as a PNG or SVG image so the LLM can visually inspect it, or optionally saves it to a local file (local mode only).

**Input**:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `shapeId` | `string` | Yes | — | Shape UUID or the special value `"selection"` for the first currently selected shape. |
| `format` | `"png"` \| `"svg"` | No | `"png"` | Output image format. |
| `mode` | `"shape"` \| `"fill"` | No | `"shape"` | `"shape"` renders the full shape tree; `"fill"` extracts the raw fill image. |
| `filePath` | `string` | No | — | Absolute path to save the export to (**local mode only**). |

**Output**: Inline image data (for the LLM to "see") or a confirmation message if saved to file.

**Behaviour**:
* Uses the Penpot Plugin API `shape.export()` for shape rendering.
* Uses `sharp` to detect and convert non-PNG pixel images to PNG before returning inline.
* The `filePath` parameter is absent from the tool schema in remote/multi-user mode.

---

### 5.5 Tool: `import_image` *(local mode only)*

**Description**: Reads a raster image from the local file system and imports it into the open Penpot design as a `Rectangle` shape with an image fill.

**Input**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | `string` | Yes | Absolute path to the image file on the local filesystem. |
| `x` | `number` | No | X position of the rectangle. |
| `y` | `number` | No | Y position of the rectangle. |
| `width` | `number` | No | Width; if only width is provided, height maintains aspect ratio. |
| `height` | `number` | No | Height; if only height is provided, width maintains aspect ratio. |

**Supported formats**: JPEG, PNG, GIF, WEBP.

**Output**: JSON object with the `shapeId` of the newly created Rectangle.

**Availability**: Only registered when `isFileSystemAccessEnabled()` is `true` (i.e. not in remote or multi-user mode).

---

### 5.6 Development REPL

**Description**: A web-based JavaScript REPL (Read–Eval–Print Loop) that allows developers to execute code snippets against the connected Penpot plugin directly from a browser.

**Access**: `http://localhost:4403/` (port configurable via `PENPOT_MCP_REPL_PORT`).

**Features**:
* Browser-based textarea input with command history (keyboard navigation).
* Displays captured console output and return values in formatted blocks.
* Executes code via the same `PluginBridge` path as the `execute_code` MCP tool.
* Intended for development and debugging only; has no authentication.

---

## 6. Operational Modes

### 6.1 Single-User Mode (Default)

* No authentication required.
* Exactly one Penpot plugin instance may be connected at a time.
* Full filesystem access enabled (import/export to files).

### 6.2 Remote Mode (`PENPOT_MCP_REMOTE_MODE=true`)

* File system access disabled (no `import_image` tool; no `filePath` on `export_shape`).
* No user token required; authentication relies on the caller's network security.
* Suitable for running the MCP server on a remote host for a single user.

### 6.3 Multi-User Mode (`--multi-user` CLI flag)

> ⚠️ Under active development; not yet production-ready.

* All HTTP sessions require a `userToken` query parameter.
* WebSocket connections require a matching `userToken` query parameter.
* Each user token maps to exactly one plugin WebSocket connection.
* File system access disabled.
* Suitable for shared/team deployments where each user has an isolated session.

---

## 7. Transport Protocols

| Protocol | Endpoint | Notes |
|----------|----------|-------|
| Streamable HTTP (MCP) | `POST/GET /mcp` | Modern MCP transport; supports multiple sessions. |
| Server-Sent Events (MCP) | `GET /sse` + `POST /messages` | Legacy MCP transport for older clients. |
| stdio (via proxy) | N/A | Supported indirectly via `mcp-remote` or similar proxies. |

---

## 8. Integration with MCP Clients

### Claude Desktop

Configure via `claude_desktop_config.json` using `mcp-remote` as a stdio proxy:

```json
{
  "mcpServers": {
    "penpot": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "http://localhost:4401/sse", "--allow-http"]
    }
  }
}
```

### Claude Code (CLI)

```shell
claude mcp add penpot -t http http://localhost:4401/mcp
```

### Any HTTP-capable MCP client

Connect directly to `http://localhost:4401/mcp` (Streamable HTTP) or `http://localhost:4401/sse` (SSE).

---

## 9. Plugin Permissions

The Penpot MCP Plugin requests the following permissions in `manifest.json`:

| Permission | Justification |
|------------|---------------|
| `content:read` | Read shapes, pages, and design elements. |
| `content:write` | Create, modify, and delete shapes. |
| `library:read` | Read shared library assets (components, tokens). |
| `library:write` | Modify library assets. |
| `comment:read` | Read design comments. |
| `comment:write` | Add or modify comments. |

---

## 10. `PenpotUtils` Utility API

The following utilities are available to LLM-authored code via the `penpotUtils` object:

| Method | Description |
|--------|-------------|
| `shapeStructure(shape, maxDepth?)` | Returns a serialisable tree of a shape and its descendants. |
| `findShapes(predicate, root?)` | Finds all shapes matching a predicate. |
| `findShape(predicate, root?)` | Finds the first shape matching a predicate. |
| `findShapeById(id)` | Finds a shape by its UUID. |
| `findPage(predicate)` | Finds a page matching a predicate. |
| `getPages()` | Returns all pages as `{ id, name }` objects. |
| `getPageById(id)` | Returns a page by UUID. |
| `getPageByName(name)` | Returns a page by name (case-insensitive). |
| `getPageForShape(shape)` | Returns the page that contains a given shape. |
| `generateCss(shape)` | Generates CSS for a shape using the Penpot CSS generator. |
| `getBounds(shape)` | Returns the true rendered bounds (text-aware). |
| `isContainedIn(child, parent)` | Checks if `child` is fully inside `parent`. |
| `setParentXY(shape, px, py)` | Sets shape position relative to its parent. |
| `addFlexLayout(container, dir)` | Adds a flex layout while preserving child visual order. |
| `analyzeDescendants(root, fn, maxDepth?)` | Applies an evaluator to all descendants; returns non-null results. |
| `base64ToByteArray(base64)` | Decodes a base64 string to `Uint8Array`. |
| `importImage(base64, mime, name, x?, y?, w?, h?)` | Imports an image as a Rectangle fill. |
| `exportImage(shape, mode, asSVG)` | Exports a shape or fill image to bytes. |
| `findTokensByName(name)` | Finds all design tokens by name. |
| `findTokenByName(name)` | Finds the first token by name. |
| `getTokenSet(token)` | Returns the token set containing a given token. |
| `tokenOverview()` | Returns a nested overview of all token sets and types. |

---

## 11. Success Metrics

| Metric | Target |
|--------|--------|
| Round-trip latency (tool call → result) | < 5 seconds for typical operations |
| Session stability | < 1 disconnection per hour under normal use |
| Tool invocation success rate | > 95% for well-formed inputs |
| Time to first design change (new user) | < 10 minutes following the README |

---

## 12. Future Roadmap

* **Production multi-user mode**: Integrate `userToken` with Penpot authentication; remove hard-coded tokens.
* **Additional tools**: Dedicated tools for reading/writing design tokens, querying comments, applying component overrides.
* **Streaming results**: For large design structures, stream partial results back to the LLM.
* **Plugin auto-update**: Detect version mismatches between server and plugin; prompt the user to update.
* **Remote file exchange**: Replace filesystem-based image import/export with a server-side upload/download mechanism to support remote mode.
* **Audit logging**: In multi-user mode, log all tool calls with user identity for compliance purposes.
