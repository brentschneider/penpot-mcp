# Penpot MCP – Security Audit Report

**Date**: 2026-03-10  
**Scope**: `mcp/` directory (server, plugin, common, REPL)  
**Auditor**: Copilot automated review

---

## Executive Summary

The Penpot MCP server is designed to allow AI language models to execute arbitrary JavaScript code inside a Penpot design project via the Penpot Plugin API. Because **arbitrary code execution is the core feature**, several findings below are inherent to the design. They are documented for awareness but are not regressions.

Three findings of note require fixes:

1. **[HIGH] XSS in REPL error output** — error messages are rendered as unescaped HTML.
2. **[MEDIUM] Duplicate token connection logic bug** — a new WebSocket connection that shares an existing token incorrectly replaces it in the routing map even after being closed, orphaning the original valid connection.
3. **[LOW] `shapeId` not sanitised in generated code** — the `shapeId` parameter in `ExportShapeTool` is interpolated into a JavaScript string without escaping.

No known CVEs were found in the current direct dependencies (`express@5.2.1`, `ws@8.19.0`, `js-yaml@4.1.1`, `sharp@0.34.5`, `pino@9.14.0`, `zod@4.3.6`).

---

## Findings

### REPL-001 · [HIGH] Cross-Site Scripting (XSS) in REPL Error Output

**File**: `packages/server/src/static/repl.html`, line 456  
**Status**: Unmitigated

**Description**:
The REPL web interface renders execution error messages from the server directly as unescaped HTML using jQuery's `.html()`. Other output (log and result) is correctly passed through the `escapeHtml()` helper, but error messages are not:

```javascript
// Vulnerable – data.error is not escaped
const errorHtml = `<div class="error-output">Error: ${data.error}</div>`;
$outputContent.html(errorHtml);
```

**Attack scenario**: If the MCP server or plugin returns an error message containing HTML/JavaScript (e.g. an exception message originating from user-controlled input processed in the plugin), the REPL UI could execute that JavaScript in the developer's browser.

**Impact**: Medium-low in practice — the REPL is not authenticated and is intended for local developer use only. However, if the REPL port (4403) is reachable from a network, a malicious Penpot design element or crafted code response could trigger XSS in anyone who opens the REPL in their browser.

**Recommendation**: Apply the same `escapeHtml()` function to `data.error` before interpolating it:

```javascript
const errorHtml = `<div class="error-output">Error: ${escapeHtml(String(data.error))}</div>`;
$outputContent.html(errorHtml);
```

---

### BRIDGE-001 · [MEDIUM] Duplicate Token Connection Overwrites Existing Client

**File**: `packages/server/src/PluginBridge.ts`, lines 64–69  
**Status**: Logic bug

**Description**:
When a new WebSocket connection arrives with a `userToken` that is already in use, the code correctly warns and closes the new connection — but the `return` statement is missing. Execution continues to line 69, which **overwrites** the original valid connection in `clientsByToken` with the now-closing new connection:

```typescript
if (this.clientsByToken.has(userToken)) {
    this.logger.warn("Duplicate connection for given user token; rejecting new connection");
    ws.close(1008, "Duplicate connection for given user token; close previous connection first.");
    // MISSING: return;  ← without this, we fall through to the set below
}

this.clientsByToken.set(userToken, connection); // ← overwrites the valid original!
```

**Impact**: After a duplicate connection attempt, the original (valid) plugin instance can no longer receive tasks — all subsequent `executePluginTask` calls will target the already-closed socket and fail with `"Plugin instance is disconnected."`. This is a denial-of-service vector in multi-user mode (any user can disrupt another user's session by attempting to connect with their token) and a reliability bug in single-user mode.

**Recommendation**: Add a `return` after closing the duplicate connection:

```typescript
if (this.clientsByToken.has(userToken)) {
    this.logger.warn("Duplicate connection for given user token; rejecting new connection");
    ws.close(1008, "Duplicate connection for given user token; close previous connection first.");
    return;
}
this.clientsByToken.set(userToken, connection);
```

---

### TOOL-001 · [LOW] Insufficient Input Sanitisation in `ExportShapeTool` Code Generation

**File**: `packages/server/src/tools/ExportShapeTool.ts`, line 92  
**Status**: Unmitigated

**Description**:
The `shapeId` parameter is interpolated directly into a JavaScript code string without escaping:

```typescript
shapeCode = `penpotUtils.findShapeById("${args.shapeId}")`;
const code = `return penpotUtils.exportImage(${shapeCode}, "${args.mode}", ${asSvg});`;
```

A `shapeId` value containing `"` followed by additional JavaScript (e.g. `"); maliciousCode; //`) would be injected into the code executed by the plugin.

**Mitigating factors**:
* The Zod schema only requires `shapeId` to be a non-empty string (`.min(1)`); no format restriction.
* In the intended workflow, the LLM reads `shapeId` values from the design and passes them back; a well-formed UUID is always used. Actual exploitation would require a maliciously crafted interaction.
* The injected code still runs inside the Penpot Plugin API sandbox, which already has `content:write` permissions — so the privilege escalation surface is limited to what the plugin can already do.

**Recommendation**: Either validate `shapeId` against a UUID regex or use `JSON.stringify` to safely embed the value:

```typescript
shapeCode = `penpotUtils.findShapeById(${JSON.stringify(args.shapeId)})`;
```

---

### DESIGN-001 · [INFORMATIONAL] Arbitrary Code Execution is By Design

**Files**: `packages/plugin/src/task-handlers/ExecuteCodeTaskHandler.ts`, `packages/server/src/tools/ExecuteCodeTool.ts`

The `execute_code` tool and REPL endpoint allow an LLM (or any caller) to run arbitrary JavaScript using `new Function()` inside the Penpot Plugin sandbox. This is the core design of the integration.

**Inherent risks**:
* Any code can call the full Penpot Plugin API, including destructive operations (e.g. deleting all shapes, overwriting fills).
* The `storage` object persists across tool calls in a session, so a prior call could pollute state for subsequent calls.
* There is no allow-list or deny-list of allowed API calls.

**Existing mitigations**:
* Code runs in the browser Penpot Plugin sandbox, which limits access to the host OS.
* File system access (read/write) is not available inside the plugin sandbox.
* In remote/multi-user mode, the `import_image` and `filePath` export options are disabled on the server side, removing server-side filesystem access.

**Recommendation**: Document the trust model clearly. Consider adding a "confirm before execution" prompt in the plugin UI for destructive operations in production deployments.

---

### DESIGN-002 · [INFORMATIONAL] No Authentication on REPL Server

**File**: `packages/server/src/ReplServer.ts`

The REPL server (port 4403) exposes a `/execute` endpoint with no authentication. Any process or browser that can reach this port can execute arbitrary code in the connected Penpot design.

**Mitigating factors**:
* The REPL is explicitly intended for development/debugging.
* The default bind address (`0.0.0.0`) means it is reachable on all interfaces; operators should restrict access with a firewall or bind to `127.0.0.1` only.

**Recommendation**: At minimum, document that the REPL port should not be exposed to untrusted networks. Consider adding a simple shared-secret header check, or disabling the REPL in production/remote mode.

---

### DESIGN-003 · [INFORMATIONAL] User Tokens Passed as URL Query Parameters

**Files**: `packages/server/src/PenpotMcpServer.ts`, `packages/plugin/src/main.ts`

In multi-user mode, `userToken` is passed as a URL query parameter (`?userToken=…`) on both HTTP and WebSocket connections. Query parameters:

* Appear in server access logs.
* Can appear in browser history and HTTP `Referer` headers.
* May be cached by intermediary proxies.

**Recommendation**: For production multi-user deployments, use an HTTP request header (e.g. `Authorization: Bearer <token>`) for the MCP HTTP transport, and use the WebSocket `Sec-WebSocket-Protocol` header or a first-frame authentication message for the WebSocket transport.

---

### DESIGN-004 · [INFORMATIONAL] No TLS / Transport Encryption

**Files**: `packages/server/src/PenpotMcpServer.ts`, `packages/plugin/src/main.ts`

All communication (MCP HTTP, WebSocket) uses plain HTTP/WS. The plugin connects using `ws://` (not `wss://`).

**Mitigating factors**:
* Local deployments (the primary use case) do not require TLS.
* The browser's Private Network Access (PNA) restrictions effectively prevent cross-origin access in Chromium-based browsers.

**Recommendation**: For remote deployments, place the servers behind a TLS-terminating reverse proxy (e.g. nginx with a Let's Encrypt certificate) and update the plugin's WebSocket URL to use `wss://`.

---

### DESIGN-005 · [INFORMATIONAL] No Rate Limiting or Request Size Limits

**File**: `packages/server/src/PenpotMcpServer.ts`, `packages/server/src/ReplServer.ts`

Neither the MCP HTTP server nor the REPL server implement rate limiting or maximum request body size limits.

**Impact**: A malicious actor could send very large code payloads or a high volume of requests, potentially causing resource exhaustion.

**Recommendation**: Add `express-rate-limit` (already in the transitive dependency tree via `@modelcontextprotocol/sdk`) and set a reasonable `express.json({ limit: '1mb' })` body size cap.

---

### DESIGN-006 · [INFORMATIONAL] Hard-Coded Token in Multi-User Plugin Build

**File**: `mcp/docs/multi-user-mode.md`

The documentation notes that the user token is "hard-coded in the plugin's source code for testing purposes" in the current multi-user mode implementation. This means all users of a multi-user build share the same token.

**Recommendation**: This is acknowledged as a known limitation. Before production use of multi-user mode, integrate with Penpot's authentication to generate per-user tokens dynamically.

---

### DESIGN-007 · [INFORMATIONAL] File System Access Not Restricted to Safe Directories

**File**: `packages/server/src/utils/FileUtils.ts`, `packages/server/src/tools/ImportImageTool.ts`, `packages/server/src/tools/ExportShapeTool.ts`

The `checkPathIsAbsolute` function only verifies that a provided path is absolute. It does not restrict paths to a designated safe directory. In local mode, an LLM could theoretically provide a sensitive absolute path (e.g. `/etc/passwd`) to `import_image`, which would read and transmit the file's contents to the plugin.

**Mitigating factors**:
* This feature is only available in local mode, where the user is running both the server and the AI client on their own machine.
* The SECURITY.md credits Ali Maharramli for identifying a prior path traversal vulnerability, suggesting this area has been reviewed.
* The Zod schema for `filePath` requires a non-empty string; it does not restrict the path further.

**Recommendation**: Consider implementing an optional allowlist of directories (e.g. `PENPOT_MCP_ALLOWED_DIRS` environment variable) that restricts file operations to specific paths.

---

## Dependency Vulnerability Scan

The following direct runtime dependencies were checked against the GitHub Advisory Database on 2026-03-10. **No known vulnerabilities were found.**

| Package | Version | Result |
|---------|---------|--------|
| `express` | 5.2.1 | ✅ No CVEs |
| `ws` | 8.19.0 | ✅ No CVEs |
| `js-yaml` | 4.1.1 | ✅ No CVEs |
| `sharp` | 0.34.5 | ✅ No CVEs |
| `pino` | 9.14.0 | ✅ No CVEs |
| `zod` | 4.3.6 | ✅ No CVEs |

---

## Summary Table

| ID | Severity | Title | Status |
|----|----------|-------|--------|
| REPL-001 | **HIGH** | XSS in REPL error output | ✅ Fixed |
| BRIDGE-001 | **MEDIUM** | Duplicate token connection overwrites existing client | ✅ Fixed |
| TOOL-001 | **LOW** | `shapeId` not sanitised in generated code | ✅ Fixed |
| DESIGN-001 | INFO | Arbitrary code execution is by design | ✅ By design |
| DESIGN-002 | INFO | No authentication on REPL server | ✅ Known (dev tool) |
| DESIGN-003 | INFO | User tokens in URL query parameters | ✅ Known limitation |
| DESIGN-004 | INFO | No TLS / transport encryption | ✅ For local use |
| DESIGN-005 | INFO | No rate limiting or request size limits | ⚠️ Enhancement |
| DESIGN-006 | INFO | Hard-coded token in multi-user mode | ✅ Known (WIP) |
| DESIGN-007 | INFO | File system access not restricted to safe directories | ⚠️ Enhancement |

---

## Recommended Actions (Priority Order)

1. **Fix BRIDGE-001** (logic bug): Add a `return` after closing the duplicate WebSocket connection in `PluginBridge.ts`. This is a straightforward one-line fix that prevents session disruption.

2. **Fix REPL-001** (XSS): Apply `escapeHtml()` to `data.error` in `repl.html`. One-line fix.

3. **Fix TOOL-001** (code injection): Use `JSON.stringify(args.shapeId)` when embedding the shape ID into the generated code string in `ExportShapeTool.ts`.

4. **Enhancement – DESIGN-005**: Add `express-rate-limit` and a body-size cap to prevent resource exhaustion.

5. **Enhancement – DESIGN-003**: Migrate token passing from query string to request headers for multi-user deployments.
