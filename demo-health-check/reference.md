# Chrome DevTools MCP — Complete Documentation
> What it is, why it exists, and how it works

**Package:** `chrome-devtools-mcp` by Google Chrome DevTools Team
**License:** Apache-2.0
**Repository:** https://github.com/ChromeDevTools/chrome-devtools-mcp

---

## Table of Contents

1. [What is chrome-devtools-mcp?](#section-1)
2. [The Problem It Solves](#section-2)
3. [Architecture Overview](#section-3)
4. [Connecting to Chrome — The Four Modes](#section-4)
5. [The Tools — Complete Reference](#section-5)
6. [The Daemon and CLI](#section-6)
7. [Configuration Reference](#section-7)
8. [Design Principles and Philosophy](#section-8)

---

<a name="section-1"></a>
## 1. What is chrome-devtools-mcp?

If you've ever wished your AI coding assistant could actually see what's happening in a browser -- click buttons, read console errors, measure page load times -- `chrome-devtools-mcp` is the bridge that makes it possible. It is a Model Context Protocol (MCP) server, built by the Google Chrome DevTools team, that gives any MCP-compatible AI assistant the ability to control and inspect a live Chrome browser. Instead of reasoning about code in isolation, your AI assistant can now observe runtime behavior, interact with authenticated pages, and gather real performance data -- all through the standardized MCP protocol.

The project is open-source under the Apache 2.0 license, published as `chrome-devtools-mcp` on npm (current version 0.20.3), and maintained in the `ChromeDevTools/chrome-devtools-mcp` GitHub repository. It requires Node.js v20.19 or newer and a current stable version of Chrome.

### Core Capabilities

At its heart, chrome-devtools-mcp exposes 29 tools across six categories. These tools give AI assistants abilities that were previously impossible without a human sitting in front of the browser:

- **Performance insights**: Record Chrome DevTools performance traces, extract Core Web Vitals (LCP, INP, CLS), and get actionable optimization suggestions -- all powered by the same trace-parsing code used in Chrome DevTools itself. CrUX field data can be fetched alongside lab measurements for a complete performance picture.
- **Browser debugging**: Take screenshots, inspect console messages with source-mapped stack traces, list and examine network requests, and run Lighthouse audits for accessibility, SEO, and best practices.
- **Reliable automation**: Built on Puppeteer, every interaction -- clicks, form fills, keyboard input, drag-and-drop -- automatically waits for the action to settle before returning. No manual sleep calls or flaky timing hacks.
- **Page understanding**: Generate accessibility tree snapshots that represent the page as structured, semantic text. Elements get stable UIDs that persist across snapshots, so the AI can reference and interact with specific elements across multiple steps.
- **Device emulation**: Simulate mobile viewports, slow network conditions (3G/4G), CPU throttling, geolocation, dark mode, and custom user agents -- all without changing browser settings permanently.
- **Memory profiling**: Capture heap snapshots for memory leak investigation, exportable as `.heapsnapshot` files for analysis with standard tooling.

### Supported AI Clients

Because chrome-devtools-mcp communicates over stdio using the MCP protocol (JSON-RPC), it works with any MCP-compatible client. The README explicitly names Gemini, Claude, Cursor, and Copilot, but any client that speaks MCP can connect. The server detects and logs which client is connected (including Gemini CLI, Claude Code, Codex, and others) for telemetry purposes, but the protocol interface is identical regardless of client.

### Quick Start

Getting started takes a single JSON configuration block in your MCP client's settings:

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["-y", "chrome-devtools-mcp@latest"]
    }
  }
}
```

Using `chrome-devtools-mcp@latest` ensures you always pull the newest version. When the AI assistant first invokes a tool, the server lazily launches a Chrome instance with a persistent user data directory at `~/.cache/chrome-devtools-mcp/chrome-profile`. From that point on, the assistant can navigate pages, take snapshots, fill forms, record performance traces, and more -- all within a single session.

For simpler use cases that don't need the full 29-tool suite, a slim mode is available:

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["-y", "chrome-devtools-mcp@latest", "--slim"]
    }
  }
}
```

Slim mode exposes just three tools -- `navigate`, `evaluate`, and `screenshot` -- reducing context window usage while still covering basic browser automation needs.

### What Makes It Different

Several design choices distinguish chrome-devtools-mcp from other browser automation MCP servers. It uses Puppeteer against real Chrome rather than a separate browser engine, which means the AI works with the same browser the developer uses -- including access to Chrome-specific APIs like the DevTools Protocol, performance tracing, and the accessibility tree. Responses are token-optimized: instead of dumping raw trace JSON or full DOM trees into the AI's context window, the server returns semantic summaries ("LCP was 3.2s, bottleneck is resource load delay") and offers `filePath` parameters for large outputs. Tools execute serially through a mutex, ensuring predictable state even when the AI issues rapid sequences of commands. And every tool call is self-healing: errors include actionable context like "Element uid '1_5' not found on page 2" rather than opaque stack traces.

### Key Takeaways

- chrome-devtools-mcp is an MCP server from Google's Chrome DevTools team that lets AI assistants control and inspect a live Chrome browser.
- It provides 29 tools spanning automation, debugging, performance analysis, network inspection, emulation, and memory profiling.
- Any MCP-compatible AI client (Gemini, Claude, Cursor, Copilot, and others) can connect with a single JSON config block.
- Responses are token-optimized -- semantic summaries and file references instead of raw data dumps -- keeping AI context windows efficient.
- A slim mode with three tools is available for simpler use cases.

---

<a name="section-2"></a>
## 2. The Problem It Solves

Modern AI coding assistants are remarkably good at reading and writing code. They can refactor a function, explain an algorithm, or generate a component from a description. But until recently, they shared a fundamental blind spot: they could not see what the code actually does when it runs in a browser. This section explains the specific gaps that chrome-devtools-mcp fills and why those gaps matter for real-world development workflows.

### The Blindfold Problem

Consider a typical debugging scenario. A user reports that a page loads slowly on mobile. A developer opens Chrome DevTools, records a performance trace, identifies that the Largest Contentful Paint is bottlenecked by a render-blocking CSS file, and restructures the loading strategy. Every step of that workflow requires a live browser -- and before chrome-devtools-mcp, AI assistants had no way to participate in any of it.

The specific gaps were:

**No live browser session.** AI assistants could reason about code statically -- reading HTML, CSS, and JavaScript files -- but could not observe the runtime result. They could not see what a page actually looked like, whether a button was clickable, or what errors appeared in the console. Asking an AI to debug a visual regression was like asking a mechanic to diagnose a car without ever starting the engine.

**The login-wall problem.** Many real-world applications sit behind authentication. A developer testing an admin dashboard, an internal tool, or a SaaS product needs to be logged in. Previous browser automation approaches launched fresh, isolated browser sessions with no cookies or saved state -- meaning the AI hit the login page and stopped. chrome-devtools-mcp solves this by connecting to a browser where the user is already authenticated. In its default mode, it uses a persistent user data directory at `~/.cache/chrome-devtools-mcp/chrome-profile` that retains cookies and sessions across runs. With Chrome 144's auto-connect feature, it can even attach to the user's running Chrome instance, with all its existing sessions intact.

**No real performance data.** AI assistants could look at code and make educated guesses about performance ("this looks like a large bundle"), but they could not measure actual Core Web Vitals. They had no access to network waterfalls, CPU profiles, memory usage, or Lighthouse scores. The difference matters: optimizing based on code inspection alone often targets the wrong bottleneck. chrome-devtools-mcp gives the AI the same trace-recording and analysis pipeline used by Chrome DevTools itself, returning semantic summaries like "LCP was 3.2s; resource load delay accounts for 40%" instead of raw data.

**Manual setup friction.** Even for developers who tried to give their AI tools browser access, the setup was painful. Connecting to Chrome's debugging port required launching the browser with special flags (`--remote-debugging-port=9222 --user-data-dir=/tmp/profile`), finding the WebSocket endpoint, and configuring the automation library. Each new session meant repeating the process. chrome-devtools-mcp reduces this to a single JSON config block -- the server handles browser launching, connection management, and lifecycle automatically.

### What Changed with Chrome 144

Chrome 144 introduced a feature that fundamentally simplified browser-AI integration: auto-connect. When a user enables remote debugging via `chrome://inspect/#remote-debugging`, Chrome writes a `DevToolsActivePort` file to its user data directory containing the debugging port and WebSocket path. The MCP server reads this file and connects automatically -- no manual port configuration, no command-line flags for the browser itself.

Critically, Chrome 144 also added a permission dialog: when an external tool attempts to connect, Chrome asks the user to explicitly allow or deny the connection. This consent mechanism means users can grant their AI assistant access to their actual browsing session -- with all its cookies, tabs, and history -- without silently exposing everything. The user stays in control.

The practical impact is significant. A developer can browse normally, encounter a bug, and ask their AI assistant to investigate -- and the assistant can connect to that exact browser session, see the same page state, read the same console errors, and record a performance trace on the actual page with all its real data. No more context switching between "the browser I use" and "the browser the AI controls."

### How It Compares to Previous Approaches

Before chrome-devtools-mcp, teams attempting browser-AI integration typically used one of two approaches, each with significant limitations.

**Playwright-only MCP servers** could automate browser interactions but operated in their own browser engine (Chromium, Firefox, or WebKit). They launched isolated sessions with no access to the user's existing browser state, couldn't record Chrome DevTools performance traces, and returned raw DOM content or screenshots that consumed enormous amounts of context window space. The AI got browser access, but it was a different browser in a different state doing different things.

**Manual remote debugging port setup** gave access to the real Chrome instance but required the developer to restart Chrome with special flags, remember the port number, and manage the connection lifecycle. The approach worked but imposed enough friction that most developers didn't bother.

chrome-devtools-mcp combines the strengths of both while avoiding their weaknesses. It uses Puppeteer against real Chrome, providing DevTools-level capabilities (performance traces, accessibility trees, Lighthouse audits, source-mapped stack traces) that pure automation tools don't expose. It connects to the user's actual browser or launches a persistent instance with retained state. And it wraps everything in the MCP protocol, so any compatible AI client can use it with zero custom integration code.

### Why This Matters for Real-World Debugging

The practical consequence is that AI coding assistants can now participate in the full debugging loop -- not just the "read code and suggest changes" portion. An assistant can navigate to a page, take an accessibility snapshot to understand the structure, record a performance trace, identify that LCP is slow because of a render-blocking resource, suggest a code change, and then re-run the trace to verify the improvement. That entire workflow happens through tool calls, with the AI reasoning about real measurements rather than static code.

For teams working on authenticated applications, internal tools, or complex SPAs where runtime behavior diverges significantly from what the source code suggests, this closes a gap that no amount of static analysis could fill.

### Key Takeaways

- AI coding assistants were blind to runtime browser behavior -- they could read code but not see what it does.
- The login-wall problem prevented AI from testing authenticated applications; persistent browser profiles and auto-connect solve this.
- Chrome 144's auto-connect feature and permission dialog enable secure, frictionless connection to a user's running browser.
- Unlike Playwright-only approaches, chrome-devtools-mcp provides DevTools-level capabilities (traces, Lighthouse, accessibility trees) against real Chrome.
- The result: AI assistants can participate in the full debug-measure-fix cycle, not just code review.

---

<a name="section-3"></a>
## 3. Architecture Overview

Understanding how chrome-devtools-mcp is built helps you reason about what it can do, what its limitations are, and how to troubleshoot when something goes wrong. The architecture follows a clean layered design: each layer has a single responsibility, and data flows in one direction from the AI assistant down through the protocol stack to Chrome and back.

### The Layer Cake

```
+--------------------------------------------------+
|  AI Assistant (Gemini, Claude, Cursor, Copilot)  |
+--------------------------------------------------+
                      | JSON-RPC over stdio
                      v
+--------------------------------------------------+
|  McpServer (@modelcontextprotocol/sdk)           |
|  - Registers 29 tools                            |
|  - Serializes execution via mutex                 |
|  - Manages telemetry (ClearcutLogger)            |
+--------------------------------------------------+
                      |
                      v
+--------------------------------------------------+
|  McpContext (central state manager)               |
|  - Page tracking (McpPage instances)             |
|  - Network/Console collectors                    |
|  - Emulation settings                            |
|  - Performance trace storage                     |
|  - Text snapshots (accessibility tree)           |
+--------------------------------------------------+
                      |
                      v
+--------------------------------------------------+
|  Puppeteer (browser automation library)          |
|  - Page, ElementHandle, BrowserContext APIs       |
+--------------------------------------------------+
                      | Chrome DevTools Protocol (CDP)
                      v
+--------------------------------------------------+
|  Chrome Browser                                   |
|  - Tabs, DOM, Network, Performance, A11y tree    |
+--------------------------------------------------+
```

Each layer communicates only with its immediate neighbors. The AI assistant never talks directly to Puppeteer, and Puppeteer never formats responses for the AI. This separation keeps the codebase maintainable and makes it possible to swap components -- a different browser automation library, a different transport, or a different AI protocol -- without rewriting everything.

### Key Components and Their Roles

**McpServer** (`index.ts`) is the entry point. Created via the `createMcpServer()` function, it instantiates a server named `chrome_devtools` using the `@modelcontextprotocol/sdk` library. Its primary responsibilities are tool registration (via `registerTool()`), connection management over `StdioServerTransport`, and telemetry logging. The server does not launch a browser at startup -- that happens lazily on the first tool call, keeping startup fast even when the AI never uses browser tools in a given session.

A critical design choice lives here: **mutex serialization**. A `toolMutex` ensures that only one tool executes at a time, with FIFO ordering. This eliminates race conditions from concurrent tool calls (which some AI clients do issue) and guarantees that the browser state is consistent between the start and end of each tool execution. The mutex is acquired before the tool handler runs and released after the response is fully serialized.

**McpContext** (`McpContext.ts`) is the central state object, created via `McpContext.from(browser, logger, options)`. It manages everything the server needs to track between tool calls:

- **Page tracking**: A `#pages` array and a `#mcpPages` WeakMap that associates each Puppeteer `Page` with an `McpPage` wrapper. One page is always designated as `#selectedPage` -- the default target for page-scoped tools.
- **NetworkCollector**: Hooks into page `request` events and maintains a rolling window of the last 3 navigations worth of network requests per page. Each request gets a stable integer ID for referencing.
- **ConsoleCollector**: Captures console messages, uncaught errors, and Chrome DevTools issues (via the CDP `Audits.enable` command and DevTools' `IssueAggregator`). Same 3-navigation rolling window.
- **Performance traces**: Stores parsed trace results from DevTools frontend trace-parsing code. Only the latest trace is kept in memory; older traces are discarded or saved to file.
- **Text snapshots**: Creates accessibility tree representations of pages using `page.accessibility.snapshot()`. Each node gets a stable UID (format: `{snapshotId}_{counter}`) that persists across snapshots through a `uniqueBackendNodeIdToMcpId` map, so the AI can reference the same element across multiple interactions.

**McpPage** (`McpPage.ts`) wraps a single Puppeteer `Page` with per-page state: the unique page ID, the latest text snapshot, emulation settings (CPU throttling, network conditions, viewport, geolocation, user agent, color scheme), an optional DevTools page reference, and the current dialog if one is open. It provides `getElementByUid(uid)` to resolve snapshot UIDs back to Puppeteer `ElementHandle` objects, which is how the AI's abstract references ("click element uid 3_12") become concrete browser actions.

### End-to-End Flow of a Tool Call

When an AI assistant asks to take a screenshot, here is exactly what happens:

1. The AI client sends a JSON-RPC `tools/call` request over stdio with `{ name: "take_screenshot", arguments: { fullPage: true } }`.
2. McpServer dispatches to the registered handler for `take_screenshot`.
3. The `toolMutex` is acquired (if another tool is running, this call waits in the FIFO queue).
4. `getContext()` is called. On the first invocation, this launches or connects to Chrome and creates the `McpContext`. On subsequent calls, it returns the existing context.
5. `detectOpenDevToolsWindows()` checks whether any Chrome DevTools panels are open (used for reading the user's selected element or network request from the DevTools UI).
6. An `McpResponse` object is created -- a builder that accumulates text lines, images, snapshots, and other response data.
7. The tool handler executes: Puppeteer calls `page.screenshot()` with the requested options.
8. `response.handle()` serializes the output. For a screenshot, this means encoding the image as base64 (or saving to a temp file if the image exceeds 2MB). It also optionally creates a text snapshot of the page, formats any recent console messages or network requests, and assembles everything into a content array.
9. The response is returned as a `CallToolResult` containing `TextContent` and `ImageContent` items.
10. The mutex is released and telemetry is logged (tool name, success/failure, bucketized latency).

The entire flow is synchronous from the AI's perspective -- it sends a request and gets back a complete response. The mutex ensures that even if the AI fires multiple tool calls in rapid succession, they execute one at a time in order.

### The Token-Optimization Strategy

AI assistants have finite context windows, and browser data is notoriously verbose. A raw Chrome performance trace can be 50,000+ lines of JSON. A full DOM tree for a modern web application can be tens of thousands of nodes. Dumping this raw data into the AI's context would be wasteful at best and would exceed token limits at worst.

chrome-devtools-mcp addresses this at every layer of the response pipeline:

**Semantic summaries over raw data.** Performance traces are parsed using the same code as Chrome DevTools' Performance panel, then returned as human-readable summaries: "LCP was 3.2s; the bottleneck is resource load delay." Network request lists show one-line summaries (`reqid=5 GET /api/users 200`) rather than full headers and bodies. Console messages are formatted concisely (`msgid=3 [error] Uncaught TypeError (2 args)`).

**File references for heavy assets.** Screenshots, performance traces, heap snapshots, and detailed network request/response bodies can be saved to files via `filePath` parameters. The response then contains just the file path, not the data itself. This is the "reference over value" design principle in action.

**Accessibility tree snapshots over DOM.** Instead of returning the full HTML DOM, `take_snapshot` returns the accessibility tree -- a semantic representation that captures the meaningful structure (headings, buttons, links, form fields) without the noise of presentational markup. A page with 10,000 DOM nodes might produce a snapshot of a few hundred lines.

**Pagination.** Network requests and console messages support `pageSize` and `pageIdx` parameters, so the AI can request just the first 10 network requests rather than all 200.

**SlimMcpResponse.** In slim mode, the response builder is replaced with `SlimMcpResponse`, which strips away all formatters, snapshots, and structured data -- returning only the bare text response lines. This produces the smallest possible responses for simple automation tasks.

### Key Takeaways

- The architecture is a clean six-layer stack: AI Client -> MCP Protocol -> McpServer -> McpContext -> Puppeteer -> Chrome.
- A mutex serializes all tool execution, ensuring consistent browser state and eliminating race conditions.
- McpContext is the central state manager, tracking pages, network requests, console messages, emulation settings, and performance traces.
- Stable UIDs on accessibility tree nodes let the AI reference the same element across multiple snapshots and tool calls.
- Token optimization is built into every response: semantic summaries, file references, accessibility trees instead of DOM, and pagination keep context windows efficient.

---

<a name="section-4"></a>
## 4. Connecting to Chrome — The Four Modes

How the MCP server connects to Chrome determines what it can access, how much setup is required, and what security trade-offs are in play. chrome-devtools-mcp supports four distinct connection modes, each designed for a different scenario. Choosing the right mode is the single most important configuration decision you'll make.

### Mode 1: Launch a New Browser (Default)

When no connection flags are specified, chrome-devtools-mcp launches a fresh Chrome instance using Puppeteer's `launch()` method with `pipe: true` for direct communication. This is the zero-configuration path -- the quick-start JSON config with no extra arguments uses this mode.

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["-y", "chrome-devtools-mcp@latest"]
    }
  }
}
```

The launched browser gets a persistent user data directory, which means cookies, saved passwords, and browsing history carry over between sessions. The directory location depends on the platform:

- **Linux/macOS**: `$HOME/.cache/chrome-devtools-mcp/chrome-profile`
- **Windows**: `%HOMEPATH%/.cache/chrome-devtools-mcp/chrome-profile`

If you specify a `--channel` other than `stable` (e.g., `canary`, `beta`, `dev`), the directory becomes `chrome-profile-{channel}` to keep channel profiles separate. The browser launches with `--hide-crash-restore-bubble` by default, and in headless mode adds `--screen-info={3840x2160}` for high-resolution rendering.

This mode is the best starting point for most users. The persistent profile means you can log into a web application once and the session persists across MCP server restarts. The trade-off is that only one instance can use this profile at a time -- launching a second server with the same profile will fail with a "browser is already running" error.

### Mode 2: Auto-Connect to Running Chrome 144+ (--autoConnect)

This is the most powerful mode for real-world debugging. Instead of launching a separate browser, the server connects to the Chrome instance you're already using -- with all its open tabs, authenticated sessions, and browsing state.

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["-y", "chrome-devtools-mcp@latest", "--auto-connect"]
    }
  }
}
```

The mechanism relies on Chrome 144's remote debugging support. When the user enables debugging via `chrome://inspect/#remote-debugging`, Chrome writes a `DevToolsActivePort` file to its user data directory. This file contains two lines: the debugging port number and the WebSocket path. The MCP server reads this file, constructs a `ws://127.0.0.1:{port}{path}` URL, and connects via `puppeteer.connect()`.

When the connection is first established, Chrome displays a permission dialog asking the user to allow or deny the incoming debugging connection. This consent step is important: once allowed, the MCP server has access to all open windows and tabs in that Chrome profile. The user must explicitly click "Allow."

You can specify which Chrome channel to connect to with `--channel` (stable, canary, beta, or dev). If you need to target a specific profile, use `--user-data-dir` to point directly at the directory containing the `DevToolsActivePort` file.

This mode conflicts with `--isolated`, `--executablePath`, and `--categoryExtensions` since those flags only apply when the server controls the browser launch.

### Mode 3: Remote Debugging Port (--browser-url)

For environments where the MCP server cannot launch Chrome directly -- sandboxed containers, remote machines, or CI systems -- you can start Chrome separately with a debugging port and point the server at it.

**Step 1**: Launch Chrome with remote debugging enabled:

```bash
chrome --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-debug-profile
```

Chrome requires a non-default user data directory when using `--remote-debugging-port` to prevent accidentally exposing the user's main profile.

**Step 2**: Configure the MCP server to connect:

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["-y", "chrome-devtools-mcp@latest", "--browser-url=http://127.0.0.1:9222"]
    }
  }
}
```

The server calls `puppeteer.connect({ browserURL: "http://127.0.0.1:9222" })`, which queries Chrome's HTTP endpoint to discover the WebSocket URL and connects.

**Security warning**: The remote debugging port has no authentication. Any application on the machine can connect to it and control the browser -- read cookies, navigate pages, execute JavaScript. Use this mode only in trusted environments, and never expose the debugging port to a network interface other than localhost.

### Mode 4: WebSocket Endpoint (--ws-endpoint)

When you need precise control over the connection -- or when connecting through an authenticated proxy -- you can specify the WebSocket URL directly.

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": [
        "-y", "chrome-devtools-mcp@latest",
        "--ws-endpoint=ws://127.0.0.1:9222/devtools/browser/<id>"
      ]
    }
  }
}
```

You can find the WebSocket endpoint by querying `http://127.0.0.1:9222/json/version` and reading the `webSocketDebuggerUrl` field from the JSON response.

This mode also supports custom headers via `--ws-headers`, which accepts a JSON string:

```json
"args": [
  "-y", "chrome-devtools-mcp@latest",
  "--ws-endpoint=ws://remote-host:9222/devtools/browser/<id>",
  "--ws-headers={\"Authorization\":\"Bearer <token>\"}"
]
```

This is the right choice for connecting to browser-as-a-service platforms, remote debugging through authenticated tunnels, or any scenario where you already have a WebSocket URL and potentially need to pass credentials.

### Isolated Mode (--isolated)

Isolated mode is not a connection mode per se but a modifier that changes how Mode 1 (launch new browser) behaves. When `--isolated` is set, the server creates a temporary user data directory that is automatically deleted when the browser closes. No cookies, passwords, or history persist between sessions.

```json
"args": ["-y", "chrome-devtools-mcp@latest", "--isolated"]
```

This is useful in two scenarios. First, when you need a clean browser state for each session -- no leftover cookies or cached data from previous runs. Second, when running multiple MCP server instances simultaneously: since each gets its own temporary profile, there's no "browser is already running" conflict.

The CLI tool (`chrome-devtools` command) defaults to `--isolated true` and `--headless true`, since CLI usage typically involves short-lived, stateless automation tasks.

### When to Use Which

| Scenario | Mode | Key flags |
|----------|------|-----------|
| First-time setup, general use | Mode 1 (launch) | *(none -- this is the default)* |
| Debug a page you're already viewing | Mode 2 (auto-connect) | `--auto-connect` |
| Sandboxed environment / CI | Mode 3 (remote port) | `--browser-url=http://...` |
| Browser-as-a-service / authenticated proxy | Mode 4 (WebSocket) | `--ws-endpoint=ws://...` |
| Clean session each time | Mode 1 + isolated | `--isolated` |
| Multiple simultaneous instances | Mode 1 + isolated | `--isolated` |
| Test on Canary/Beta/Dev channel | Mode 1 or 2 | `--channel=canary` |

### Key Takeaways

- Mode 1 (launch) is the default and requires zero configuration -- the server manages the entire Chrome lifecycle with a persistent profile.
- Mode 2 (auto-connect) is the most powerful for real-world debugging, connecting to the user's running Chrome 144+ instance with all its existing state.
- Mode 3 (browser-url) and Mode 4 (ws-endpoint) are for advanced scenarios: sandboxed environments, remote machines, and authenticated connections.
- Isolated mode creates a throwaway profile for clean-slate sessions and enables running multiple instances in parallel.
- The remote debugging port (Mode 3) has no built-in authentication -- treat it as a security-sensitive surface.

---

<a name="section-5"></a>
## 5. The Tools — Complete Reference

The tools are what make chrome-devtools-mcp useful. Each tool is a discrete, composable operation that does one thing predictably -- a design principle the project calls "small, deterministic blocks." Rather than offering a single "test this page" mega-tool, the server exposes 29 focused tools that AI assistants chain together into workflows: navigate to a URL, take a snapshot to understand the page, fill a form, click a button, check the console for errors, record a performance trace.

All tools are organized into six categories. Categories can be individually disabled via CLI flags (`--no-category-emulation`, `--no-category-performance`, `--no-category-network`) to reduce the tool surface when certain capabilities aren't needed.

### Input Tools (9)

These tools let the AI interact with page elements. Every input tool that targets an element uses a `uid` parameter -- the stable identifier assigned to elements in accessibility tree snapshots (from `take_snapshot`). Most input tools accept an optional `includeSnapshot` parameter that automatically takes a new snapshot after the action completes, so the AI can see the result without a separate tool call.

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| **click** | Click on an element identified by UID | `uid` (required), `dblClick`, `includeSnapshot` |
| **drag** | Drag one element onto another | `from_uid`, `to_uid` (both required), `includeSnapshot` |
| **fill** | Type into an input/textarea or select an option from a `<select>` | `uid`, `value` (both required), `includeSnapshot` |
| **fill_form** | Fill multiple form elements in a single call | `elements` (array of `{uid, value}`), `includeSnapshot` |
| **handle_dialog** | Accept or dismiss a browser dialog (alert/confirm/prompt) | `action` ("accept" or "dismiss"), `promptText` |
| **hover** | Hover over an element | `uid` (required), `includeSnapshot` |
| **press_key** | Press a key or key combination | `key` (required, e.g., "Enter", "Control+A"), `includeSnapshot` |
| **type_text** | Type text into the currently focused element | `text` (required), `submitKey` (e.g., "Enter", "Tab") |
| **upload_file** | Upload a file through a file input element | `uid`, `filePath` (both required), `includeSnapshot` |

A typical interaction workflow looks like: `take_snapshot` to discover elements and their UIDs, then `fill` to enter text into a field, then `click` on the submit button, then `take_snapshot` again to verify the result. The `fill_form` tool streamlines this by accepting multiple field values in one call.

The `click` tool uses `waitForEventsAfterAction` internally, meaning it waits for any triggered navigation or DOM mutation to settle before returning. This eliminates the need for manual waits between clicking a link and observing the result.

### Navigation Tools (6)

These tools manage pages (tabs) and navigation within them.

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| **navigate_page** | Go to a URL, or navigate back/forward/reload | `type` (url/back/forward/reload), `url`, `ignoreCache`, `timeout`, `initScript` |
| **new_page** | Open a new tab and load a URL | `url` (required), `background`, `isolatedContext`, `timeout` |
| **close_page** | Close a page by its ID (the last page cannot be closed) | `pageId` (required) |
| **list_pages** | List all open pages with their IDs and URLs | *(none)* |
| **select_page** | Set a page as the active context for subsequent tools | `pageId` (required), `bringToFront` |
| **wait_for** | Wait for specific text to appear on the page | `text` (array of strings), `timeout` |

The page model works like browser tabs: `new_page` opens a tab, `select_page` switches focus, and all page-scoped tools operate on the currently selected page. The `navigate_page` tool is versatile -- beyond loading URLs, it supports browser back/forward history navigation, cache-bypassing reloads, and an `initScript` parameter that injects JavaScript before any page script runs (useful for mocking APIs or setting up test fixtures). The `wait_for` tool uses Puppeteer's `Locator.race` with both `aria/` and `text/` locators across all frames, providing robust text detection.

### Emulation Tools (2)

Emulation tools simulate different device conditions on the selected page without affecting the browser's global settings.

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| **emulate** | Set device emulation conditions | `networkConditions` (Offline/Slow 3G/Fast 3G/Slow 4G/Fast 4G), `cpuThrottlingRate` (1-20), `geolocation`, `userAgent`, `colorScheme` (dark/light/auto), `viewport` |
| **resize_page** | Resize the browser window dimensions | `width`, `height` (both required) |

The `emulate` tool is particularly powerful for performance testing. Setting `networkConditions` to "Slow 3G" and `cpuThrottlingRate` to 4 simulates a low-end mobile device, letting the AI observe how a page performs under real-world constraints. The viewport parameter accepts a compact format string: `"375x812x3,mobile,touch"` sets width, height, device pixel ratio, and enables mobile/touch emulation in one value. All emulation settings are stored per-page in the `McpPage` object and persist until explicitly changed. They're also automatically restored after Lighthouse audits (which temporarily override them).

### Performance Tools (4)

These tools provide access to Chrome's performance tracing and memory profiling capabilities -- the same data you'd see in Chrome DevTools' Performance panel.

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| **performance_start_trace** | Begin recording a performance trace | `reload` (default: true), `autoStop` (default: true), `filePath` |
| **performance_stop_trace** | Stop an active trace recording | `filePath` |
| **performance_analyze_insight** | Get details on a specific performance insight | `insightSetId`, `insightName` (both required, e.g., "LCPBreakdown") |
| **take_memory_snapshot** | Capture a heap snapshot for memory analysis | `filePath` (required, `.heapsnapshot` extension) |

The performance trace workflow is typically: call `performance_start_trace` (which reloads the page by default and auto-stops after 5 seconds), then examine the returned summary containing Core Web Vitals, insight names, and CrUX field data. For deeper analysis, call `performance_analyze_insight` with a specific insight like "DocumentLatency" or "LCPBreakdown" to get a detailed breakdown. Traces can be saved to `.json.gz` files via the `filePath` parameter for later analysis in Chrome DevTools.

The trace recording uses the same comprehensive category set as Chrome DevTools and Lighthouse, including `devtools.timeline`, `blink.user_timing`, `v8.cpu_profiler`, and `loading`. Parsed results include CrUX (Chrome User Experience Report) field data when available, providing real-world performance context alongside lab measurements.

### Network Tools (2)

Network tools inspect HTTP traffic captured during page navigation.

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| **list_network_requests** | List requests since the last navigation | `pageSize`, `pageIdx`, `resourceTypes` (filter by type), `includePreservedRequests` |
| **get_network_request** | Get full details of a specific request | `reqid`, `requestFilePath`, `responseFilePath` |

The `NetworkCollector` hooks into Puppeteer's page `request` events and maintains a rolling window of the last 3 navigations. Each request gets a stable integer ID (`reqid`). The `list_network_requests` tool returns concise one-line summaries (`reqid=5 GET /api/users 200`), while `get_network_request` provides full headers, response bodies, redirect chains, and failure information. Large request/response bodies can be saved to files via `requestFilePath` and `responseFilePath` parameters -- body content is limited to 10,000 characters when returned inline.

If Chrome DevTools is open, `get_network_request` can also read the currently selected request from the DevTools Network panel (by omitting the `reqid` parameter), bridging the gap between what the developer is looking at and what the AI can inspect.

### Debugging Tools (6)

These tools provide general inspection and analysis capabilities.

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| **take_snapshot** | Create an accessibility tree snapshot of the page | `verbose`, `filePath` |
| **take_screenshot** | Capture a visual screenshot | `format` (png/jpeg/webp), `quality`, `uid` (element), `fullPage`, `filePath` |
| **evaluate_script** | Execute a JavaScript function in the page | `function` (required), `args` (UIDs resolved to elements) |
| **list_console_messages** | List console messages since last navigation | `pageSize`, `pageIdx`, `types` (including "issue") |
| **get_console_message** | Get full details of a specific console message | `msgid` (required) |
| **lighthouse_audit** | Run a Lighthouse audit for a11y, SEO, best practices | `mode` (navigation/snapshot), `device` (desktop/mobile), `outputDirPath` |

`take_snapshot` is the primary tool for understanding page content. It returns an accessibility tree where each node includes a UID, role, name, and state attributes (disabled, expanded, focused, selected). This structured representation is far more useful for AI reasoning than raw HTML -- a heading hierarchy, landmark regions, and interactive elements are immediately apparent. The `verbose` flag includes nodes that the accessibility tree normally filters out (decorative images, hidden elements).

`evaluate_script` accepts a JavaScript function declaration string and optional `args` that are UIDs -- these UIDs are automatically resolved to their corresponding DOM `ElementHandle` objects before the function executes. This lets the AI write JavaScript that operates on specific page elements identified from a snapshot.

`lighthouse_audit` runs Google Lighthouse directly through Puppeteer, covering accessibility, SEO, and best-practices categories (performance is excluded since the dedicated performance tools provide better trace-based analysis). Results can be saved to a directory as full HTML reports.

### Slim Mode (3 Tools)

When started with `--slim`, the server exposes only three simplified tools:

| Tool | Full-mode equivalent | Simplification |
|------|---------------------|----------------|
| **navigate** | navigate_page | URL-only (no back/forward/reload), fixed 30s timeout |
| **evaluate** | evaluate_script | Raw script string (no function declaration, no args/UID resolution) |
| **screenshot** | take_screenshot | PNG only, always saves to temp file (no format/quality/element options) |

Slim mode uses `SlimMcpResponse` instead of the full `McpResponse`, returning only bare text content with no snapshots, network data, or console messages. This dramatically reduces response payload size, making it suitable for AI clients with tight context limits or for simple scraping and testing tasks that don't need the full DevTools feature set.

### Key Takeaways

- 29 tools across 6 categories cover the full range of browser interaction: input, navigation, emulation, performance, network, and debugging.
- Tools are composable building blocks: `take_snapshot` for understanding, `click`/`fill` for interaction, `take_screenshot` for visual verification, `performance_start_trace` for measurement.
- Stable UIDs from accessibility tree snapshots are the primary mechanism for referencing page elements across tool calls.
- Slim mode reduces the surface to 3 essential tools for lightweight automation scenarios.
- Categories can be individually disabled to reduce the tool surface when certain capabilities aren't needed.

---

<a name="section-6"></a>
## 6. The Daemon and CLI

Most users interact with chrome-devtools-mcp through an AI assistant's MCP client, but the project also ships a standalone CLI that lets you control Chrome directly from the terminal. This is useful for scripting, quick debugging without an AI assistant, and CI/CD pipelines. The CLI is powered by a daemon architecture that keeps the browser alive between commands, enabling stateful workflows without restarting Chrome for every operation.

### Why the Daemon Exists

The MCP server is designed to run as a long-lived process: it launches Chrome on the first tool call and keeps it running for subsequent calls. But CLI commands are short-lived by nature -- you type a command, get a result, and return to the shell prompt. Without a daemon, every CLI command would need to launch a new Chrome instance, wait for it to start, execute the tool, and then shut everything down. That's slow and stateless -- you couldn't click a button in one command and take a screenshot in the next, because the browser would be gone between them.

The daemon solves this by decoupling the browser lifecycle from the CLI command lifecycle. A background process owns the browser and the MCP server, and CLI commands connect to it, execute a tool, and disconnect -- leaving the browser running for the next command.

### How the Daemon Works (daemon.ts)

When the daemon starts, it performs three setup steps:

1. **Writes a PID file** to `{runtimeHome}/daemon.pid` so other processes can discover whether a daemon is already running.
2. **Spawns the MCP server as a child process** using `StdioClientTransport`. The daemon acts as an MCP client, connecting to its own child MCP server via stdio. This means the daemon uses the exact same MCP server code that AI clients use -- there's no separate implementation.
3. **Opens a socket server** for CLI commands to connect to:
   - **Linux/macOS**: A Unix socket at `$XDG_RUNTIME_DIR/chrome-devtools-mcp/server.sock` or `/tmp/chrome-devtools-mcp-{uid}.sock`
   - **Windows**: A named pipe at `\\.\pipe\chrome-devtools-mcp\server.sock`

The socket server accepts three message types:

- **`invoke_tool`**: Forwards a tool call to the MCP server via `mcpClient.callTool()` and returns the result. This is how CLI commands execute tools.
- **`stop`**: Triggers a graceful shutdown -- closes the MCP client, transport, and server, then removes the PID file.
- **`status`**: Returns daemon metadata: PID, socket path, start date, version, and the arguments it was started with.

The daemon listens for `SIGTERM`, `SIGINT`, and `SIGHUP` signals for graceful shutdown, ensuring Chrome is properly closed and the PID file is cleaned up.

### How the Client Connects (client.ts)

The client module provides three key functions:

**`startDaemon(mcpArgs)`** spawns the daemon process as a detached child with `stdio: 'ignore'`, then polls for the PID file to appear (indicating the daemon is ready). The MCP server arguments are serialized and passed through to the daemon, so flags like `--headless` or `--browser-url` propagate correctly.

**`sendCommand(command)`** opens a socket connection to the daemon, sends a single JSON command via `PipeTransport`, waits for the response (with a 60-second timeout), and disconnects. Each CLI command creates a fresh connection, executes, and closes -- the socket is not held open between commands.

**`stopDaemon()`** sends a `stop` command and waits for the PID file to be removed, confirming that the daemon has fully shut down.

### The CLI Entry Point (chrome-devtools.ts)

The CLI is built with yargs and exposes subcommands for daemon management plus all available tools:

```bash
# Daemon management
chrome-devtools start          # Start the daemon (stops existing one first)
chrome-devtools status         # Check if daemon is running
chrome-devtools stop           # Stop the daemon and close Chrome

# Tool commands (daemon auto-starts if not running)
chrome-devtools navigate_page --url https://example.com
chrome-devtools take_snapshot
chrome-devtools click --uid 3_5
chrome-devtools take_screenshot --filePath ./screenshot.png
chrome-devtools performance_start_trace
```

A key convenience: if you run a tool command and no daemon is running, the CLI automatically starts one. This means you can go straight to `chrome-devtools navigate_page --url https://example.com` without an explicit `start` step.

The CLI sets different defaults than the MCP server mode: `--headless true` and `--isolated true` are the defaults for CLI usage, since CLI automation tasks typically don't need a visible browser window and benefit from clean-slate sessions. It also sets `--viaCli` and `--experimentalStructuredContent` internally.

### A Typical CLI Workflow

Here's a practical example of using the CLI to debug a slow page:

```bash
# Navigate to the page (daemon auto-starts with headless Chrome)
chrome-devtools navigate_page --url https://myapp.example.com/dashboard

# Take a snapshot to understand the page structure
chrome-devtools take_snapshot

# Record a performance trace
chrome-devtools performance_start_trace

# Examine the results (auto-stops after 5 seconds)
# The trace summary is printed to stdout

# Get details on a specific insight
chrome-devtools performance_analyze_insight \
  --insightSetId "0" \
  --insightName "LCPBreakdown"

# Take a screenshot for visual reference
chrome-devtools take_screenshot --filePath ./dashboard.png

# Done -- stop the daemon
chrome-devtools stop
```

Each command connects to the same daemon, which maintains the same browser session throughout. The page state, navigation history, and collected network/console data all persist across commands.

### Response Handling

When the CLI receives a `CallToolResult` from the daemon, `handleResponse()` formats it for terminal output. Text content is printed directly to stdout. Images are saved to temporary files, and the file path is printed so you can open them. JSON output is available for machine consumption, making it straightforward to pipe CLI results into other tools or scripts.

### Key Takeaways

- The daemon is a background process that keeps Chrome alive between CLI commands, enabling stateful multi-step workflows.
- It spawns the same MCP server used by AI clients as a child process -- there's no separate CLI-specific implementation.
- CLI commands connect to the daemon via a Unix socket (Linux/macOS) or named pipe (Windows), execute a tool, and disconnect.
- The daemon auto-starts when you run a tool command, so explicit `start` is rarely needed.
- CLI defaults differ from MCP server defaults: headless and isolated are both true, optimized for automation rather than interactive debugging.

---

<a name="section-7"></a>
## 7. Configuration Reference

Getting chrome-devtools-mcp configured correctly is the difference between a seamless debugging experience and a frustrating "why won't it connect?" session. This section is a complete reference for every CLI flag and environment variable the server accepts, organized by function so you can find what you need quickly.

All flags use camelCase as the canonical form (e.g., `--browserUrl`) but also accept kebab-case (e.g., `--browser-url`). Boolean flags can be negated with `--no-` prefix (e.g., `--no-headless`, `--no-usage-statistics`).

### Browser Connection Flags

These flags control how the server connects to Chrome. They are mutually exclusive -- use only one connection method at a time.

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--autoConnect` / `--auto-connect` | boolean | `false` | Auto-connect to a running Chrome 144+ instance. Requires remote debugging enabled via `chrome://inspect/#remote-debugging`. |
| `--browserUrl` / `--browser-url` / `-u` | string | -- | HTTP URL of a running debuggable Chrome instance (e.g., `http://127.0.0.1:9222`). |
| `--wsEndpoint` / `--ws-endpoint` / `-w` | string | -- | WebSocket endpoint URL (e.g., `ws://127.0.0.1:9222/devtools/browser/<id>`). Conflicts with `--browserUrl`. |
| `--wsHeaders` / `--ws-headers` | string (JSON) | -- | Custom headers for WebSocket connection (e.g., `'{"Authorization":"Bearer token"}'`). Requires `--wsEndpoint`. |

### Browser Launch Flags

These flags control how Chrome is launched when using the default launch mode (Mode 1).

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--headless` | boolean | `false` | Run Chrome in headless mode (no visible window). CLI defaults to `true`. |
| `--executablePath` / `--executable-path` / `-e` | string | -- | Path to a custom Chrome executable. |
| `--isolated` | boolean | `false` | Create a temporary user data directory, cleaned up on close. CLI defaults to `true`. |
| `--userDataDir` / `--user-data-dir` | string | -- | Path to a custom user data directory. Overrides the default persistent profile. |
| `--channel` | string | `stable` | Chrome channel to use: `stable`, `canary`, `beta`, or `dev`. |
| `--viewport` | string | -- | Initial viewport size (e.g., `1280x720`). |
| `--proxyServer` / `--proxy-server` | string | -- | Proxy server configuration passed to Chrome (e.g., `http://proxy:8080`). |
| `--acceptInsecureCerts` / `--accept-insecure-certs` | boolean | `false` | Ignore SSL/TLS certificate errors. Useful for testing against self-signed certificates. |
| `--chromeArg` / `--chrome-arg` | array | -- | Additional Chrome launch arguments (e.g., `--chrome-arg=--disable-extensions`). Can be specified multiple times. |
| `--ignoreDefaultChromeArg` / `--ignore-default-chrome-arg` | array | -- | Disable specific default Chrome arguments that the server normally passes. Can be specified multiple times. |

### Tool Category Flags

These flags control which categories of tools are exposed to the AI client. Disabling unused categories reduces the tool surface and simplifies the AI's decision space.

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--categoryEmulation` / `--category-emulation` | boolean | `true` | Include emulation tools (`emulate`, `resize_page`). |
| `--categoryPerformance` / `--category-performance` | boolean | `true` | Include performance tools (`performance_start_trace`, `performance_stop_trace`, `performance_analyze_insight`, `take_memory_snapshot`). |
| `--categoryNetwork` / `--category-network` | boolean | `true` | Include network tools (`list_network_requests`, `get_network_request`). |
| `--categoryExtensions` / `--category-extensions` | boolean | `false` | Include extension management tools (`install_extension`, `uninstall_extension`, `list_extensions`, `reload_extension`, `trigger_extension_action`). Hidden by default. |
| `--slim` | boolean | `false` | Expose only 3 simplified tools (`navigate`, `evaluate`, `screenshot`). Overrides all category flags. |

### Performance and Data Flags

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--performanceCrux` / `--performance-crux` | boolean | `true` | Fetch CrUX (Chrome User Experience Report) field data when recording performance traces. Sends trace URLs to the Google CrUX API. |

### Privacy and Telemetry Flags

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--usageStatistics` / `--usage-statistics` | boolean | `true` | Enable usage statistics collection (tool invocations, latency, environment info). Sent to Google Clearcut. |
| `--logFile` / `--log-file` | string | -- | Path to a debug log file. Set the `DEBUG=*` environment variable for verbose output. |

### Experimental Flags

These flags enable features that are not yet stable. They may change or be removed in future versions.

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--experimentalScreencast` / `--experimental-screencast` | boolean | `false` | Enable video recording tools (`screencast_start`, `screencast_stop`). Requires ffmpeg installed. |
| `--experimentalPageIdRouting` / `--experimental-page-id-routing` | boolean | `false` | Add a `pageId` parameter to all page-scoped tools, allowing explicit page targeting without `select_page`. |
| `--experimentalDevtools` / `--experimental-devtools` | boolean | `false` | Enable DevTools target automation. |
| `--experimentalVision` / `--experimental-vision` | boolean | `false` | Enable vision-based tools (`click_at` for coordinate-based clicking). |
| `--experimentalStructuredContent` / `--experimental-structured-content` | boolean | `false` | Output structured JSON content alongside text responses. |
| `--experimentalIncludeAllPages` / `--experimental-include-all-pages` | boolean | `false` | Include all page-like targets (service workers, shared workers) in page listing. |
| `--experimentalInteropTools` / `--experimental-interop-tools` | boolean | `false` | Enable interoperability tools (`get_tab_id`). |

### Environment Variables

| Variable | Effect |
|----------|--------|
| `CHROME_DEVTOOLS_MCP_NO_USAGE_STATISTICS` | When set (any value), disables usage statistics collection. Equivalent to `--no-usage-statistics`. |
| `CI` | When set, automatically disables usage statistics collection. Standard CI environment variable. |
| `DEBUG` | Controls debug logging verbosity. Set to `*` for all debug output, or `mcp:*` for MCP-specific logs. Requires `--logFile` to write to a file. |
| `DISPLAY` | On Linux/UNIX, the server checks this variable to determine if a display is available for non-headless mode. |

### Node.js Version Requirements

The server requires Node.js `^20.19.0 || ^22.12.0 || >=23`. This means:

- Node.js 20.x: version 20.19.0 or newer
- Node.js 21.x: **not supported**
- Node.js 22.x: version 22.12.0 or newer
- Node.js 23.x and above: any version

### Example Configurations

**Basic setup (default launch mode):**
```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["-y", "chrome-devtools-mcp@latest"]
    }
  }
}
```

**Connect to running Chrome with telemetry disabled:**
```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": [
        "-y", "chrome-devtools-mcp@latest",
        "--auto-connect",
        "--no-usage-statistics"
      ]
    }
  }
}
```

**Headless with custom viewport and no emulation tools:**
```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": [
        "-y", "chrome-devtools-mcp@latest",
        "--headless",
        "--viewport=1920x1080",
        "--no-category-emulation"
      ]
    }
  }
}
```

**Remote browser through authenticated WebSocket:**
```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": [
        "-y", "chrome-devtools-mcp@latest",
        "--ws-endpoint=ws://remote:9222/devtools/browser/abc123",
        "--ws-headers={\"Authorization\":\"Bearer mytoken\"}"
      ]
    }
  }
}
```

### Key Takeaways

- All flags accept both camelCase and kebab-case forms; boolean flags can be negated with `--no-` prefix.
- Connection flags (`--autoConnect`, `--browserUrl`, `--wsEndpoint`) are mutually exclusive -- pick one connection method.
- Tool category flags let you trim the exposed tool surface; `--slim` overrides everything with just 3 tools.
- Usage statistics are on by default but can be disabled via flag or environment variable; they're auto-disabled in CI.
- Node.js 21.x is not supported -- the version constraint is `^20.19.0 || ^22.12.0 || >=23`.

---

<a name="section-8"></a>
## 8. Design Principles and Philosophy

The design of chrome-devtools-mcp is shaped by a specific challenge: browser data is vast, AI context windows are finite, and the gap between "raw browser output" and "useful AI input" is where most browser-automation tools fall short. The project's `docs/design-principles.md` codifies seven principles that guide every architectural and API decision. Understanding these principles explains not just how the tools work, but why they work the way they do.

### 1. Agent-Agnostic API

**Principle**: Use standards like MCP. Don't lock in to one LLM. Interoperability is key.

chrome-devtools-mcp communicates exclusively through the Model Context Protocol over stdio using JSON-RPC. It does not import any AI provider's SDK, does not assume a particular model's capabilities, and does not format responses for a specific client's UI. The same server binary works identically with Gemini, Claude, Cursor, Copilot, Codex, and any future MCP-compatible client.

This isn't just an abstract commitment to openness -- it has concrete implementation consequences. The server's telemetry detects which client is connected (via user-agent and connection patterns), but the tool interface, response format, and behavior are identical across all of them. The `McpServer` is instantiated with a generic name (`chrome_devtools`) and communicates through `StdioServerTransport`, the most universal MCP transport available. There are no client-specific code paths, no conditional response formatting, and no feature flags tied to a particular AI provider.

### 2. Token-Optimized

**Principle**: Return semantic summaries. "LCP was 3.2s" is better than 50k lines of JSON. Files are the right location for large amounts of data.

This principle drives more implementation decisions than any other. Every response pipeline in the codebase is designed to minimize token consumption while maximizing information density.

Performance traces are parsed using Chrome DevTools frontend code and returned as structured summaries: Core Web Vitals scores, identified insights (like "LCPBreakdown" or "DocumentLatency"), and CrUX field data comparisons -- not raw trace events. Network requests appear as one-line summaries (`reqid=5 GET /api/users 200`) in list view, with full details available on demand via `get_network_request`. Console messages follow the same pattern: concise list format with detailed drill-down. The accessibility tree snapshot (`take_snapshot`) represents pages as semantic structure rather than DOM, compressing thousands of HTML nodes into hundreds of meaningful lines.

For data that can't be meaningfully summarized -- screenshots, full trace files, heap snapshots, large request/response bodies -- the server uses file references. The `filePath` parameter on tools like `take_screenshot`, `performance_stop_trace`, and `take_memory_snapshot` saves data to disk and returns the path. The AI gets a pointer, not a payload. Response bodies over 10,000 characters are truncated inline but can be saved in full via `responseFilePath`. Images over 2MB are automatically saved to temp files rather than being returned as base64.

Slim mode (`--slim`) takes this to its logical extreme: `SlimMcpResponse` strips away all formatters, snapshots, network data, and console data, returning only bare text content.

### 3. Small, Deterministic Blocks

**Principle**: Give agents composable tools (Click, Screenshot), not magic buttons. Each tool does one thing predictably.

The 29-tool surface is deliberately granular. There is no "test this page" tool, no "debug this performance issue" tool, no "fill out this form and submit it" tool. Instead, there are atomic operations -- `click`, `fill`, `take_snapshot`, `performance_start_trace` -- that the AI chains together into workflows.

This design trusts the AI to be a capable planner. Rather than encoding workflow logic into the tool server (which would be brittle and opinionated), the server provides reliable building blocks and lets the AI decide the sequence. A performance debugging workflow might be: `navigate_page` -> `performance_start_trace` -> `performance_analyze_insight` -> `evaluate_script` -> `take_screenshot`. A form-testing workflow might be: `take_snapshot` -> `fill_form` -> `click` -> `wait_for` -> `take_snapshot`. The server doesn't need to know about these patterns -- it just needs each individual tool to behave predictably.

Predictability is key. `click` always waits for the action to settle (via `waitForEventsAfterAction`). `take_snapshot` always returns the accessibility tree with stable UIDs. `performance_start_trace` with `autoStop: true` always records for 5 seconds then stops. No hidden state, no surprising side effects, no "it depends" behavior.

### 4. Self-Healing Errors

**Principle**: Return actionable errors that include context and potential fixes.

When something goes wrong, the error message should tell the AI what to do next -- not just what failed. The codebase follows this consistently:

- Element not found: `"Element uid '1_5' not found on page 2."` -- tells the AI which UID failed and on which page, suggesting it should re-take a snapshot to get updated UIDs.
- No page selected: `"No page selected. Call list_pages to see open pages."` -- directly tells the AI the next step.
- Browser not connected: errors include the connection mode that was attempted and suggest checking the relevant flag or Chrome configuration.

This principle exists because AI assistants cannot interactively debug their own tool failures the way a human developer can. They need enough context in the error message itself to formulate a recovery plan. A generic "element not found" error forces the AI to guess; an error that includes the UID, the page, and an implicit suggestion to refresh the snapshot gives it a clear path forward.

### 5. Human-Agent Collaboration

**Principle**: Output must be readable by machines (structured) AND humans (summaries). The response format includes both text and structured JSON content.

The `McpResponse.format()` method builds two parallel representations of every response: a `text` string for human-readable display and a `structuredContent` JSON object for programmatic consumption. The text version uses natural language summaries, indented accessibility tree formatting, and readable network/console log formats. The structured version provides typed data that an AI client's UI can render as tables, trees, or interactive elements.

This dual-format approach also shows up in the DevTools integration. When Chrome DevTools is open, the server reads the user's selected element and selected network request from the DevTools UI (via `getDevToolsData()`), marking them in the snapshot output. This bridges what the human is looking at with what the AI is analyzing -- both are working from the same context.

The snapshot formatter illustrates this well: each line reads naturally (`uid=3_5 button "Submit Order" focused`) while also being machine-parseable. The AI can extract the UID for a subsequent `click` call, and a human reading the same output immediately understands the page structure.

### 6. Progressive Complexity

**Principle**: Tools should be simple by default but offer advanced optional arguments for power users.

Every tool in the codebase works with minimal arguments. `take_snapshot` needs zero parameters -- call it and get a complete page representation. `take_screenshot` needs zero parameters -- it captures the current viewport as PNG. `performance_start_trace` with no arguments reloads the page, records for 5 seconds, and returns a summary with Core Web Vitals.

But each of these tools also accepts optional parameters that unlock advanced behavior. `take_snapshot` accepts `verbose` (include normally-filtered nodes) and `filePath` (save to disk for large pages). `take_screenshot` accepts `format`, `quality`, `uid` (screenshot a specific element), `fullPage`, and `filePath`. `performance_start_trace` accepts `reload` (skip the reload), `autoStop` (keep recording indefinitely), and `filePath` (save the raw trace).

The `emulate` tool is a particularly clear example: every parameter is optional. Call it with just `networkConditions: "Slow 3G"` to throttle the network, or combine six parameters to simulate a specific device profile. The API never forces you to specify things you don't care about.

This design reduces the cognitive load on AI assistants. A simple prompt like "take a screenshot" maps directly to a zero-argument tool call. A complex prompt like "take a full-page screenshot of just the sidebar element as JPEG at 80% quality" maps to the same tool with additional parameters filled in.

### 7. Reference over Value

**Principle**: For heavy assets (screenshots, traces, videos), return a file path or resource URI, never the raw data stream.

This principle complements token optimization but addresses a different concern: transport efficiency and client compatibility. A 5MB heap snapshot or a 50MB performance trace cannot be streamed through a JSON-RPC text response. Even base64-encoded screenshots can overwhelm clients with limited message size handling.

The implementation is consistent: `take_memory_snapshot` requires a `filePath` parameter (there is no inline option -- heap snapshots are always saved to disk). `performance_stop_trace` optionally saves to a `.json.gz` file. `take_screenshot` saves to a temp file automatically when the image exceeds 2MB. `get_network_request` offers `requestFilePath` and `responseFilePath` for saving large bodies.

The one exception, noted in the design principles document itself, is that some MCP clients have built-in support for displaying images directly. For these clients, screenshots under 2MB are returned as inline base64 `ImageContent` -- the right trade-off between convenience and size.

### Key Takeaways

- The seven principles are not abstract ideals -- each one drives specific, observable implementation decisions throughout the codebase.
- Token optimization is the most impactful principle: semantic summaries, file references, accessibility trees, pagination, and slim mode all serve it.
- Agent-agnostic design through MCP means the server works identically across all AI clients with zero client-specific code.
- Progressive complexity ensures tools are simple to use by default but powerful when needed -- no mandatory configuration for common operations.
- Self-healing errors tell the AI what to do next, not just what went wrong, enabling autonomous recovery from failures.

---

*Documentation produced by agent team `chrome-devtools-mcp-docs`*
*Source: https://github.com/ChromeDevTools/chrome-devtools-mcp*
