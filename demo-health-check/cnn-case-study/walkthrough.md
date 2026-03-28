# CNN.com Health Check — Complete Walkthrough
## A Deep-Dive into chrome-devtools-mcp, Step by Step

**Target site:** https://www.cnn.com
**Date:** 2026-03-27
**Purpose:** This document is a complete, annotated record of every action taken
to produce the CNN health report — what each tool is, what it does, how it
works, what went in and what came out. By the end you will understand not just
what we found, but *why* we used each tool and what you would use it for in
your own projects.

---

## What is chrome-devtools-mcp?

`chrome-devtools-mcp` is a **Model Context Protocol (MCP) server** built by
Google's Chrome DevTools team. It gives an AI assistant (like Claude Code) the
ability to control and inspect a real Chrome browser using the **Chrome
DevTools Protocol (CDP)** — the same protocol that powers the DevTools panel
you open with F12.

Without this MCP server, an AI assistant can only read and write text files. With
it, the AI gains 29 tools that let it open pages, take screenshots, read console
errors, intercept network requests, run performance traces, and run Lighthouse
audits — all through natural language prompts.

**How the connection works:**

```
You (typing prompts)
    ↓
AI assistant (Claude Code)
    ↓  calls MCP tools
chrome-devtools-mcp server (running via npx)
    ↓  speaks Chrome DevTools Protocol (CDP) over WebSocket
Chrome browser (a real tab on your machine)
    ↓  returns data
chrome-devtools-mcp server
    ↓  formats and returns results
AI assistant → interprets and explains to you
```

The MCP server is launched automatically by Claude Code when you start a session.
It connects to Chrome, opens tabs, and streams back everything DevTools can see.

---

## The 29 Tools at a Glance

chrome-devtools-mcp provides exactly 29 tools, grouped into 5 categories:

| Category | Tools |
|----------|-------|
| **Navigation** | `list_pages`, `new_page`, `select_page`, `close_page`, `navigate_page` |
| **Interaction** | `click`, `hover`, `drag`, `fill`, `fill_form`, `type_text`, `press_key`, `upload_file`, `handle_dialog`, `wait_for` |
| **Inspection** | `take_snapshot`, `take_screenshot`, `evaluate_script`, `list_console_messages`, `get_console_message`, `list_network_requests`, `get_network_request` |
| **Performance** | `performance_start_trace`, `performance_stop_trace`, `performance_analyze_insight`, `take_memory_snapshot` |
| **Audit** | `lighthouse_audit`, `emulate`, `resize_page` |

In this CNN health check, we exercised **11 of these 29 tools**. The remaining
18 are described at the end of this document.

---

## Overview: The 6-Step Health Check Process

| Step | Goal | Tools Used |
|------|------|-----------|
| 1 | Open the page and handle the consent dialog | `new_page`, `wait_for`, `take_snapshot`, `take_screenshot`, `evaluate_script` |
| 2 | Understand the page structure visually | `take_snapshot`, `take_screenshot` |
| 3 | Collect runtime debug data | `list_console_messages`, `list_network_requests` |
| 4 | Run a Lighthouse audit | `lighthouse_audit` (timed out — see notes) |
| 5 | Record a performance trace | `performance_start_trace`, `performance_stop_trace`, `performance_analyze_insight` |
| 6 | Synthesise into a report | (no new tools — AI writes the report) |

---

## Step 1 — Open the Page

### Objective

Navigate Chrome to `https://www.cnn.com` and get it into a clean, fully-loaded
state that the remaining tools can inspect. This sounds trivial, but CNN presents
a **consent dialog** ("Legal Terms and Privacy") that blocks the entire page —
no content is visible and the accessibility tree is completely empty until it is
dismissed.

This step demonstrates the combination of navigation tools, inspection tools,
and JavaScript execution to handle real-world obstacles that a site puts in
front of automated visitors.

---

### Tool: `new_page`

**What it is:** A navigation tool that opens a new browser tab and loads a URL.

**What it does:** Instructs Chrome via CDP to create a new tab, navigate it to
the specified URL, and wait for the initial page load to complete.

**How it works internally:** The MCP server sends a CDP `Target.createTarget`
command to open a new tab, then `Page.navigate` to load the URL. Chrome fires
a `Page.loadEventFired` event when the initial HTML, CSS, and synchronous
JavaScript have finished loading. The tool resolves at that point.

**Input:**
```json
{ "url": "https://www.cnn.com" }
```

**Output:**
```
## Pages
1: about:blank
2: https://www.cnn.com/ [selected]
```
The output lists all open tabs and marks the newly created one as `[selected]`.
Subsequent tools automatically operate on the selected tab.

**What it told us:** The tab opened and CNN started loading. The `[selected]`
marker is important — all inspection tools (screenshot, snapshot, console, etc.)
operate on the currently selected page. If you have multiple tabs open, you
use `select_page` to switch between them.

---

### Tool: `wait_for`

**What it is:** A synchronisation tool that pauses execution until specified text
appears on the page.

**What it does:** Polls the page's visible content on a short interval and
resolves as soon as any of the provided strings appears. If the timeout is
reached before the text appears, it throws an error.

**How it works internally:** The MCP server repeatedly queries the page's
accessibility tree (the same source as `take_snapshot`) looking for matching
text. This is more reliable than a fixed sleep because it resolves exactly when
the content is ready, not before and not long after.

**Input:**
```json
{
  "text": ["CNN", "News", "Watch"],
  "timeout": 15000
}
```
We provided three alternative strings (`"CNN"`, `"News"`, `"Watch"`) — the tool
resolves when *any* of them appears. The timeout of 15,000ms (15 seconds) is the
maximum wait time.

**Output:**
```
Element matching one of ["CNN","News","Watch"] found.
```
Followed by the latest accessibility tree snapshot.

**What it told us:** The text appeared quickly, but the snapshot still showed
almost no content — just `RootWebArea` and a live region. This was our first
clue that something was blocking the page. A few seconds after navigation, content
should be richly visible in the accessibility tree.

---

### Tool: `take_snapshot`

**What it is:** An inspection tool that captures the page's **accessibility tree**
as structured text.

**What it does:** Reads the browser's internal accessibility representation of
the page — the same data used by screen readers, browser extensions, and
automated testing tools. Every interactive element, heading, image, button, and
text node appears in this tree with a unique identifier (`uid`) that other tools
can reference to click, hover, or fill elements.

**How it works internally:** CDP exposes the accessibility tree via the
`Accessibility` domain. The MCP server calls `Accessibility.getFullAXTree`,
processes the result into a readable hierarchical format, and assigns stable `uid`
values to each node. These `uid`s are used as input to interaction tools like
`click` and `fill`.

**Input:**
```json
{ "verbose": true }
```
The `verbose: true` flag requests every available detail for each node
(labels, states, roles, descriptions). Without it, only the most important
fields are returned.

**Output (what we actually got — a warning sign):**
```
uid=1_0 RootWebArea "Breaking News, Latest News and Videos | CNN"
  uid=3_0 ignored
    uid=3_1 generic
      uid=3_2 ignored
      uid=3_3 ignored
      ... (hundreds of "ignored" nodes)
```

**What it told us:** Hundreds of `ignored` nodes in the accessibility tree means
the consent dialog is marking the entire page content as `aria-hidden="true"` —
a standard accessibility pattern where a modal dialog hides the background from
screen readers. This confirmed we had to dismiss the dialog before any other
inspection would work.

**Note on `take_snapshot` vs `take_screenshot`:** These are complementary, not
interchangeable. The snapshot gives you *structured semantic data* — you can
find a button by its label and click it programmatically. The screenshot gives
you a *visual image* — you can see what the user actually sees. The snapshot is
more useful for automation; the screenshot is more useful for confirming visual
state.

---

### Tool: `take_screenshot`

**What it is:** An inspection tool that captures the visible viewport as an image.

**What it does:** Renders the current state of the selected tab and returns
a PNG, JPEG, or WebP image, either inline or saved to a file.

**How it works internally:** CDP's `Page.captureScreenshot` command triggers a
full render of the viewport and returns the image as a base64-encoded string.
The MCP server decodes this and either attaches it to the response or saves it
to disk.

**Input (first call, checking state):**
```json
{}
```
No arguments — take a screenshot of the current viewport, return it inline.

**Input (second call, saving the final screenshot):**
```json
{ "filePath": "health-report/screenshot.png" }
```

**Output:** A rendered image of the page at the moment of capture.

**What we saw:** The consent dialog was front and centre — the CNN logo, the
"Legal Terms and Privacy" heading, and an "Agree" button. The page content was
visible but greyed out behind it.

This confirmed what the snapshot hinted at: the dialog was blocking everything.
Screenshots are often the fastest way to understand *why* a snapshot looks wrong.

---

### Tool: `evaluate_script`

**What it is:** An inspection and interaction tool that executes arbitrary
JavaScript inside the page and returns the result.

**What it does:** Runs a JavaScript function you provide in the context of the
current page — with full access to `document`, `window`, the DOM, and any
JavaScript globals the page has defined. The return value of the function is
serialised as JSON and returned to the AI.

**How it works internally:** CDP's `Runtime.callFunctionOn` command executes the
function in the page's JavaScript context. The result is serialised using the
V8 engine's structured-clone algorithm and returned as a JSON value.

**Why we needed it here:** The consent dialog's "Agree" button did not appear
in the accessibility tree (it was inside the consent manager's shadow DOM or
styled in a way that made it inaccessible to `take_snapshot`). The `click` tool
requires a `uid` from a snapshot — but since all nodes were `ignored`, there
were no valid `uid`s to click.

`evaluate_script` bypasses this limitation entirely — it runs in the page's own
JavaScript context, so it can find and click any element regardless of whether
it appears in the accessibility tree.

**We made three attempts:**

**Attempt 1 — look for a button with text "Agree":**
```javascript
() => {
  const buttons = [...document.querySelectorAll('button')];
  const agreeBtn = buttons.find(b => b.innerText.trim() === 'Agree');
  if (agreeBtn) { agreeBtn.click(); return 'clicked Agree'; }
  return 'buttons: ' + buttons.map(b => b.innerText.trim()).join(' | ');
}
```
Result: `"button not found, buttons: Cancel | Submit | Close icon | Subscribe | Sign in | Confirm My Choices"` — no button labelled exactly "Agree".

**Attempt 2 — click "Confirm My Choices":**
```javascript
() => {
  const buttons = [...document.querySelectorAll('button')];
  const confirm = buttons.find(b => b.innerText.includes('Confirm'));
  if (confirm) { confirm.click(); return 'clicked: ' + confirm.innerText.trim(); }
}
```
Result: `"clicked: Confirm My Choices"` — but the dialog did not dismiss.
The consent manager requires its own "Agree" link, not the granular choices button.

**Attempt 3 — find any element (including `<a>` tags) with "Agree" text:**
```javascript
() => {
  const all = [...document.querySelectorAll('button, a, [role="button"]')];
  const matches = all.filter(el =>
    el.textContent.trim().toLowerCase().includes('agree')
  );
  matches.forEach(el => el.click());
  return matches.map(el => el.tagName + ': ' + el.textContent.trim());
}
```
Result: `["A: Agree"]` — the "Agree" element was an `<a>` anchor tag, not a
`<button>`. Clicking it dismissed the dialog.

**What this teaches:** Real websites use non-standard patterns. The `evaluate_script`
tool is your escape hatch when the accessibility tree does not expose what you
need. It is also invaluable for reading JavaScript state, checking feature flags,
or measuring DOM metrics that no other tool surfaces.

**After the dialog was dismissed**, a final `take_screenshot` confirmed the page
was fully loaded and the CNN homepage was visible.

---

## Step 2 — Understand the Page Structure

### Objective

With the page loaded, build a picture of what CNN's homepage contains —
its structure, key interactive elements, and any immediate accessibility signals.
This step gives the AI (and you) enough context to know what you're looking at
before diving into runtime data.

---

### Tool: `take_snapshot` (second use)

After dismissing the consent dialog, `take_snapshot` was called again. This time
the accessibility tree populated fully, revealing:

- The page title: *"Breaking News, Latest News and Videos | CNN"*
- Navigation bar: US, World, Politics, Business, Health, More, Watch, Subscribe,
  Sign In
- Live updates ticker: War in Iran, Tiger Woods, Kash Patel, Savannah Guthrie,
  'No Kings' rallies, Elite Eight, Fire weather
- Lead story: *"House GOP rejects Senate-approved DHS bill, likely extending
  shutdown"*
- Secondary stories in a grid layout below the fold

**Key capability demonstrated:** The accessibility tree gives the AI a
*token-efficient, semantic understanding* of the page. Instead of processing
thousands of pixels in an image, the AI receives structured text that says
"this is a heading", "this is a navigation landmark", "this link goes here".
This is why `take_snapshot` is preferred over `take_screenshot` for automation —
it is faster, cheaper, and more actionable.

---

### Tool: `take_screenshot` (second use — saving to file)

**Input:**
```json
{ "filePath": "health-report/screenshot.png" }
```

**Output:** Saved `screenshot.png` to the `health-report/` directory.

This screenshot serves two purposes in the report:
1. Visual proof of the page state at the time of analysis
2. A reference image that non-technical readers can understand at a glance

**Other available options for `take_screenshot`:**
- `fullPage: true` — captures the entire scrollable page, not just the visible
  viewport
- `format: "jpeg"` or `"webp"` — smaller file sizes (JPEG/WebP vs PNG)
- `quality: 80` — compression level for JPEG/WebP
- `uid: "3_42"` — capture only a specific element identified in a snapshot

---

## Step 3 — Collect Runtime Debug Data

### Objective

Capture the live runtime data that only exists while the page is running in a
real browser: console messages (errors, warnings, deprecation notices) and
network requests (every HTTP request the page made, with status codes,
types, and sizes). This is data that no static analysis tool or spider can
collect — you need a real browser.

---

### Tool: `list_console_messages`

**What it is:** An inspection tool that retrieves everything written to the
browser's JavaScript console since the last navigation.

**What it does:** Returns console messages of any type: `log`, `info`, `warn`,
`error`, `debug`, and more — including messages from third-party scripts, browser
internals, and framework warnings. Each message includes its type, content, and
the number of arguments passed.

**How it works internally:** CDP's `Console` and `Runtime` domains capture
console output. The MCP server collects all messages since the last
`Page.navigate` event and returns them as a paginated list. Source-mapped stack
traces are included where available.

**Input:**
```json
{ "types": ["error", "warn"] }
```
We filtered to only `error` and `warn` to focus on problems. Without the
`types` filter, `log` and `info` messages from CNN's extensive analytics would
have buried the important signals.

**Additional options:**
- `pageSize: 50` — return at most 50 messages per call
- `pageIdx: 1` — fetch the second page of results
- `includePreservedMessages: true` — include messages from the previous 3
  navigations (useful for debugging redirects)

**Output (10 messages, all errors or warnings):**

| ID | Type | Message |
|----|------|---------|
| 137 | warn | `overflow: visible` on img/video may produce visual content outside element bounds |
| 141 | error | Failed to load resource: 410 Gone |
| 142 | error | Failed to load resource: 410 Gone |
| 148 | error | Attestation check for Protected Audience on `openwebmp.com` failed |
| 168 | warn | `[GPT]` Using deprecated `googletag.encryptedSignalProviders` |
| 173 | warn | iframe with `allow-scripts` + `allow-same-origin` can escape its sandbox |
| 174–175 | warn | iFrameResizer timeout on health widget (×2) |
| 183–186 | warn | DRM robustness level not specified (×2, from video player) |

**What this told us:**
- **Two 410 Gone errors** — CNN is requesting resources that no longer exist on
  its own servers. These are stale references from a previous deploy that were
  not cleaned up.
- **Sandbox escape** — An ad or tracking iframe has both `allow-scripts` and
  `allow-same-origin` set simultaneously. This combination allows the iframe to
  break out of its sandbox and access the parent page's DOM and cookies. This is
  a real security concern with ad-tech iframes.
- **Deprecated Google Ad Manager API** — `encryptedSignalProviders` was
  deprecated by Google. CNN's ad integration needs to migrate to
  `secureSignalProviders`.
- **DRM warnings** — The video player is initialising DRM (Digital Rights
  Management) without specifying a robustness level, which can lead to
  inconsistent playback behaviour across devices.

---

### Tool: `list_network_requests`

**What it is:** An inspection tool that retrieves every HTTP request the page
made since the last navigation, with full metadata.

**What it does:** Returns a list of all network requests including URL, HTTP
method, status code, resource type, transfer size, timing, and which script
initiated the request. This reveals the page's complete dependency graph —
every image, script, stylesheet, API call, tracking pixel, and ad request.

**How it works internally:** CDP's `Network` domain intercepts all network
activity. The MCP server collects every `Network.requestWillBeSent` and
`Network.responseReceived` event since the last navigation and assembles them
into a list. Each entry includes both request and response metadata.

**Input:**
```json
{ "pageSize": 200 }
```
We requested up to 200 entries. CNN made well over 200 requests — the full
dataset was so large (194,000+ characters) that the MCP server automatically
saved it to a temporary file rather than returning it inline.

**Additional options:**
- `resourceTypes: ["script", "fetch"]` — filter to only JavaScript files and
  AJAX calls
- `resourceTypes: ["image"]` — filter to only images (useful for finding
  unoptimised images)
- `includePreservedRequests: true` — include requests from the previous 3
  navigations

**Output:** A JSON array with one entry per request. Each entry contains:
- URL
- HTTP status code
- Resource type (script, image, fetch, stylesheet, etc.)
- Transfer size in bytes
- Timing (queued, sent, download complete, processing complete)
- Initiator chain (which script caused this request)
- Response headers including Cache-Control

**What this told us (from the performance trace analysis that followed):**
- CNN made **200+ network requests** in a single page load
- The full third-party payload exceeded **8MB** of JavaScript alone
- 52+ distinct third-party domains were contacted
- The top transfers: Wunderkind 2.5MB, Google/Doubleclick 1.1MB, Integral Ad
  Science 705KB, Optimizely 652KB
- Many resources were cached for only 5–10 minutes despite having content-hash
  filenames that make them safe to cache for a year

**The companion tool — `get_network_request`:**
The `list_network_requests` tool gives you the list. The companion tool
`get_network_request` (not used in this run) fetches the **full details of a
single request** including the complete request and response headers and body.
Use it when you need to inspect a specific request in depth — for example,
checking what authentication headers an API call sends, or reading the full
error body from a failed request.

---

## Step 4 — Lighthouse Audit

### Objective

Run Google Lighthouse — the industry-standard web quality auditing tool — to get
scored reports on **Accessibility**, **SEO**, **Best Practices**, and
**Performance**. Lighthouse produces a structured report with specific findings
(e.g. "12 images are missing alt text", "the page is missing a meta description")
rather than raw metrics.

---

### Tool: `lighthouse_audit`

**What it is:** An audit tool that runs a full Google Lighthouse analysis on the
current page and returns structured scores and findings.

**What it does:** Lighthouse runs a battery of checks across four categories
(Accessibility, SEO, Best Practices, Performance), scores each category 0–100,
and produces specific findings with explanations and links to remediation guides.

**How it works internally:** Lighthouse is embedded in Chrome DevTools. The MCP
server triggers it via CDP's `Audits` domain. Lighthouse either reloads the page
in a throttled environment (`navigation` mode) or analyses the current page state
without reloading (`snapshot` mode), runs all audit checks, and returns a JSON
report. The MCP server formats this as readable summaries and optionally saves
the full HTML report to disk.

**Input (attempt 1 — navigation mode):**
```json
{
  "device": "desktop",
  "mode": "navigation",
  "outputDirPath": "health-report"
}
```
- `device: "desktop"` — emulate a desktop browser (alternative: `"mobile"`)
- `mode: "navigation"` — reload the page in a throttled network environment and
  audit the fresh load (more accurate for performance)
- `outputDirPath` — save the full HTML report to this directory

**Input (attempt 2 — snapshot mode):**
```json
{
  "device": "desktop",
  "mode": "snapshot",
  "outputDirPath": "health-report"
}
```
- `mode: "snapshot"` — audit the current page state without reloading (faster,
  but less accurate for performance metrics)

**Output (what we hoped for):** Lighthouse category scores (0–100) and specific
findings per category, plus an HTML report file saved to `health-report/`.

**What actually happened — timeout:**
Both attempts timed out with:
```
Network.emulateNetworkConditions timed out.
```

**Why:** Lighthouse works by emulating network conditions (throttling bandwidth
to simulate a mobile connection). On CNN — with its 200+ requests, 8MB+ of
third-party JavaScript, and dozens of ad-tech calls — this emulation took longer
than the MCP server's protocol timeout allowed.

**What this teaches:** `lighthouse_audit` is the right tool for most sites.
For exceptionally heavy sites with massive ad-tech stacks, it may time out.
When it does, the performance trace tools (`performance_start_trace` /
`performance_stop_trace` / `performance_analyze_insight`) provide deeper
performance data, and the combination of `list_console_messages` and
`list_network_requests` provides much of what the Accessibility and Best
Practices audits would catch.

**When `lighthouse_audit` works well:**
- Normal websites, e-commerce sites, documentation sites, SaaS products
- Sites without aggressive ad-tech or dozens of third-party scripts
- When you want a single scored report with specific actionable findings

---

## Step 5 — Performance Trace

### Objective

Record what the browser's engine actually did during a real page load — every
network request, JavaScript execution, style recalculation, layout reflow, and
paint operation, mapped against a millisecond timeline. Extract **Core Web
Vitals** (the metrics Google uses to score your site in search rankings) and
identify the specific functions and scripts causing performance problems.

This is the most powerful step in the entire health check. No other tool gives
you this level of precision about *what* is slow and *exactly which line of code*
is causing it.

---

### Tool: `performance_start_trace`

**What it is:** A performance tool that begins recording a Chrome DevTools
performance trace on the current page.

**What it does:** Instructs Chrome to start capturing a detailed trace of
everything happening in the browser — network requests, JavaScript call stacks,
style recalculations, layout operations, paint events, and GPU compositing — and
optionally reloads the page so the trace captures the complete load sequence
from the very first network request.

**How it works internally:** CDP's `Tracing` domain starts a trace session.
Chrome begins recording timestamped events for every significant operation
across the network stack, V8 JavaScript engine, Blink rendering engine, and GPU
compositor. The MCP server optionally triggers a page reload immediately after
starting the trace, so the trace captures the full page load from byte zero.

**Input:**
```json
{
  "reload": true,
  "autoStop": true,
  "filePath": "health-report/trace.json"
}
```
- `reload: true` — reload the page so the trace starts from the very beginning
  of the load
- `autoStop: true` — automatically stop recording when the page finishes loading
  (rather than requiring a manual `performance_stop_trace` call)
- `filePath` — save the raw trace data to disk (raw traces are large JSON files
  that can be loaded in chrome://tracing or Chrome DevTools)

**What happened:** The first attempt timed out (10,000ms navigation timeout —
CNN was slow to reload under trace conditions). The trace session was left open.
A `performance_stop_trace` call was needed to close it before trying again.

**Second attempt (without filePath, with longer implicit timeout):**
```json
{
  "reload": true,
  "autoStop": true
}
```
This succeeded. CNN reloaded under trace conditions and the trace captured the
complete page load sequence.

---

### Tool: `performance_stop_trace`

**What it is:** A performance tool that stops the active trace recording and
returns the results.

**What it does:** Ends the trace session started by `performance_start_trace`,
processes the raw trace data, and returns a structured summary including:
- Core Web Vitals (LCP, CLS, INP) — both lab-measured and field data from
  real users (CrUX)
- A list of available "insights" — named findings about specific performance
  problems detected in the trace
- Bounds of each navigation in the trace (start/end timestamps)

**How it works internally:** CDP's `Tracing.end` command stops the trace.
Chrome flushes all buffered events. The MCP server processes the raw trace JSON,
applies Chrome's DevTools performance analysis algorithms, computes Core Web
Vitals, and identifies which named insights apply to this trace.

**Input:**
```json
{}
```
No arguments needed — stops whichever trace is currently running.

**Output (key sections):**

**Core Web Vitals — Lab measurements (this specific page load):**
```
LCP: 1,556 ms
CLS: 0.01
```

**Core Web Vitals — Field data (real users, CrUX p75):**
```
LCP: 1,547 ms
INP: 145 ms
CLS: 0.04
```

**Available insights (7 identified in this trace):**

| Insight | Description |
|---------|-------------|
| `LCPBreakdown` | Breakdown of the 4 LCP phases |
| `CLSCulprits` | Elements causing layout shifts |
| `NetworkDependencyTree` | Chain of blocking network requests |
| `FontDisplay` | Font loading strategy causing invisible text |
| `DOMSize` | DOM element count and depth |
| `ThirdParties` | Third-party script performance cost |
| `ForcedReflow` | JavaScript causing synchronous layout |
| `Cache` | Resources with inefficient cache policies |

Each insight is a named finding that can be drilled into using
`performance_analyze_insight`.

**Understanding the Core Web Vitals:**

| Metric | Full Name | What it measures | Good threshold |
|--------|-----------|-----------------|---------------|
| **LCP** | Largest Contentful Paint | How long until the main content is visible | < 2,500ms |
| **CLS** | Cumulative Layout Shift | How much the page "jumps" as it loads | < 0.1 |
| **INP** | Interaction to Next Paint | How quickly the page responds to clicks | < 200ms |

**CNN's results:** All three metrics are in the "Good" range. CNN's infrastructure
is genuinely fast. The problems are hidden in the weight of the ad-tech stack,
which doesn't hurt the headline metrics but does consume enormous resources.

---

### Tool: `performance_analyze_insight`

**What it is:** A performance tool that provides deep analysis of one specific
named insight from a completed trace.

**What it does:** Takes one of the insight names returned by
`performance_stop_trace` and returns detailed findings: which specific scripts,
elements, or resources are involved, how much time or bytes are wasted, and links
to remediation guides.

**How it works internally:** Chrome DevTools' performance insights engine applies
specialised analysis algorithms to the raw trace data for each insight type.
The MCP server calls these algorithms on demand and formats the results.

**Input structure:**
```json
{
  "insightSetId": "NAVIGATION_0",
  "insightName": "ThirdParties"
}
```
- `insightSetId` — identifies which navigation in the trace to analyse
  (a trace can cover multiple navigations)
- `insightName` — one of the insight names from `performance_stop_trace` output

**We called this tool 4 times**, once per insight:

---

#### Insight 1: `ThirdParties`

**Input:** `{ "insightSetId": "NAVIGATION_0", "insightName": "ThirdParties" }`

**Output summary:**

CNN loads scripts from **52+ distinct third-party vendors**.

Top transfer sizes:
| Vendor | Size | Purpose |
|--------|------|---------|
| Wunderkind | 2.5 MB | Behavioural targeting |
| Google/Doubleclick | 1.1 MB | Ad serving |
| Integral Ad Science | 705 KB | Ad fraud detection |
| Optimizely | 652 KB | A/B testing |
| Piano | 594 KB | Paywall/subscriptions |
| Outbrain | 589 KB | Content recommendations |
| Amazon Ads | 453 KB | Ad serving |

Top main-thread execution times:
| Vendor | Main Thread Time |
|--------|----------------|
| cnn.io (first-party CDN) | 3,052 ms |
| script.ac | 2,209 ms |
| Integral Ad Science | 1,233 ms |
| Wunderkind | 1,178 ms |
| Google/Doubleclick | 1,041 ms |

**Significance:** The main thread is the browser's single thread for running
JavaScript and performing layout. While it is busy executing ad-tech code, it
cannot respond to user interactions. Over 10,000ms of combined third-party
main-thread execution — on a single page load — is extraordinary.

---

#### Insight 2: `LCPBreakdown`

**Input:** `{ "insightSetId": "NAVIGATION_0", "insightName": "LCPBreakdown" }`

**Output:**
```
LCP element: H2.container__title_url-text (the main headline — a text node)
Total LCP time: 1,556 ms

Phase breakdown:
  Time to First Byte (TTFB):    19 ms  (1.2%)
  Element render delay:      1,537 ms  (98.8%)
```

**What this means:**
- CNN's server responds in 19ms — excellent.
- The headline (`H2`) is in the HTML from the server — it does not need to be
  fetched from a network.
- But the browser cannot *paint* the headline until 1,537ms have passed.
- That delay is caused by render-blocking scripts that run before the browser
  is allowed to paint.

In other words: CNN's server is fast, but the ad-tech layer is holding the
browser hostage for 1.5 seconds before the user sees the headline.

---

#### Insight 3: `ForcedReflow`

**Input:** `{ "insightSetId": "NAVIGATION_0", "insightName": "ForcedReflow" }`

**Output:**
```
Total reflow time: 3,906 ms

Top contributors (selected):
  adsafeprotected r()         2,386 ms
  boltPlayer Fc()             1,470 ms
  boltPlayer getSize()          622 ms
  outbrain.js c.FM()            818 ms
  tinypass.min.js i()           632 ms
  cxense cx.js                  356 ms
  outbrain.js Ug()              444 ms
```

**What is a forced reflow?**

A forced reflow (also called a forced synchronous layout) happens when JavaScript
*reads* a layout property (like `element.offsetWidth`, `element.getBoundingClientRect()`,
or `element.scrollHeight`) *immediately after* making a change to the DOM or styles.

The browser normally batches layout recalculations and does them once per frame.
When JavaScript reads a layout property after a DOM change, it forces the browser
to stop everything, recalculate the layout immediately, and then return the value.
This is called "layout thrashing" when it happens repeatedly.

```javascript
// ❌ Forces a reflow on every iteration — 3,906ms wasted on CNN
elements.forEach(el => {
  el.style.width = el.offsetWidth + 'px';  // read after write = forced reflow
});

// ✅ Batch reads before writes — zero forced reflows
const widths = elements.map(el => el.offsetWidth);      // all reads first
elements.forEach((el, i) => { el.style.width = widths[i] + 'px'; }); // all writes after
```

On CNN's 5,874-node DOM, each forced reflow requires the browser to measure
positions for thousands of elements. The ad verification scripts
(`adsafeprotected.com`) are the worst offenders — they call `getBoundingClientRect()`
inside loops to verify that ad slots are actually visible.

---

#### Insight 4: `Cache`

**Input:** `{ "insightSetId": "NAVIGATION_0", "insightName": "Cache" }`

**Key findings:**

| Resource | Current TTL | Problem |
|----------|------------|---------|
| `cnnsans-regular.woff2` | **0 seconds** | Font re-downloaded on every visit |
| `cnnsans-bold.woff2` | **0 seconds** | Font re-downloaded on every visit |
| `cnnsans-italic.woff2` | **0 seconds** | Font re-downloaded on every visit |
| `registry.api.cnn.io/bundles/fave/ui-34be6e59/ui` | 600s | Has a content hash in URL — could be 1 year |
| `registry.api.cnn.io/bundles/fave/vendor-f9ea2c98/vendor` | 600s | Has a content hash in URL — could be 1 year |
| All `media.cnn.com` images | 300s | News images, 5-min cache is reasonable but short |
| `zion-web-client.min.js` | 0s | Should have a TTL |
| `tag.wknd.ai/340/i.js` | 60s | Marketing tag, should be longer |

**What this means:** Every time a user visits CNN, their browser re-downloads the
custom CNN fonts from scratch. The fonts are hundreds of kilobytes and never change
— they should be cached for at least a year. Similarly, the JS bundles use
content-hash filenames (the hash in `ui-34be6e59` changes when the file changes),
making them safe to cache indefinitely — but CNN only caches them for 10 minutes.

---

#### Insight 5: `DOMSize`

**Input:** `{ "insightSetId": "NAVIGATION_0", "insightName": "DOMSize" }`

**Output:**
```
Total elements: 5,874
Maximum DOM depth: 43 nodes
Most children: 49 (on BODY element)

Largest layout events:
  Layout update: 69ms (5,042 of 5,042 nodes needing layout)
  Layout update: 55ms (5,043 of 5,043 nodes needing layout)
  Style recalculation: 114ms (1,090 elements)
  Style recalculation: 63ms (650 elements)
  Style recalculation: 60ms (2,290 elements)
```

Google's recommended maximum is 1,500 DOM elements. CNN has **5,874** — nearly
4× the recommendation. The consequence is visible in the layout event durations:
when a forced reflow triggers, the browser must measure positions for all 5,042
nodes that need layout — taking 55–69ms per event.

The DOM size and forced reflow problems multiply each other: a larger DOM makes
each forced reflow more expensive.

---

## Step 6 — Synthesise the Report

### Objective

Combine all findings from steps 1–5 into a single structured document that:
1. Presents findings in order of severity
2. Explains the root cause of each problem
3. Provides concrete, actionable recommendations
4. Is readable by both technical developers and non-technical stakeholders

### No new tools

Step 6 uses no new MCP tools. The AI reads all the data collected in steps 1–5
and writes the report as markdown. This is the synthesis step — the AI's role is
to connect the dots across all the tool outputs.

**For example:** The AI connects:
- LCPBreakdown (1,537ms render delay) ← caused by →
- ThirdParties (10,000ms of third-party scripts executing on the main thread)

And expresses this as: *"CNN's server is fast (19ms TTFB), but the ad-tech layer
blocks the browser from rendering the headline for 1.5 seconds."*

**Output:** `health-report/report.md` — the full health report with Core Web
Vitals table, third-party analysis, forced reflow findings, cache audit, console
error table, and top 5 prioritised recommendations.

---

## Tool Coverage Summary

### Tools used in this health check (11 of 29)

| Tool | Category | Used in Step | Purpose |
|------|----------|-------------|---------|
| `new_page` | Navigation | 1 | Open CNN in a new tab |
| `wait_for` | Interaction | 1 | Wait for CNN content to appear |
| `take_snapshot` | Inspection | 1, 2 | Read accessibility tree; detect consent dialog |
| `take_screenshot` | Inspection | 1, 2 | Visual confirmation; save screenshot to file |
| `evaluate_script` | Inspection | 1 | Dismiss consent dialog via JavaScript |
| `list_console_messages` | Inspection | 3 | Collect errors and warnings |
| `list_network_requests` | Inspection | 3 | Collect all HTTP requests |
| `lighthouse_audit` | Audit | 4 | Attempt Lighthouse audit (timed out on CNN) |
| `performance_start_trace` | Performance | 5 | Begin recording performance trace |
| `performance_stop_trace` | Performance | 5 | Stop trace; get Core Web Vitals + insight list |
| `performance_analyze_insight` | Performance | 5 | Deep-dive on 4 specific insights |

**Coverage: 11 / 29 tools = 38%**

A health check of this type naturally exercises the inspection and performance
categories most heavily. The interaction and navigation categories are more
relevant for testing user flows (login, checkout, form submission).

---

### Tools NOT used — and when you would use them

#### Navigation tools (unused: 3 of 5)

**`list_pages`**
Lists all currently open tabs with their URLs and IDs. Use this at the start of
a session to see what tabs are already open, or to find the tab ID of a specific
page you want to switch to.

**`select_page`**
Switches the "selected" tab — the tab that all subsequent tools operate on. Use
this when you have multiple tabs open and need to inspect a specific one. For
example: open your app in tab 1, open a competing site in tab 2, then switch
between them to compare.

**`close_page`**
Closes a tab by its ID. Use this for cleanup after automated workflows, or to
close pop-up windows opened by the page under test.

**`navigate_page`**
Navigates the *current* tab to a new URL (or goes back, forward, or reloads).
Unlike `new_page`, this does not open a new tab — it reuses the existing one.
Also supports `ignoreCache: true` (force reload from server, bypassing browser
cache) and `initScript` (inject a JavaScript snippet that runs before any other
script on the new page — powerful for mocking APIs or disabling features before
the page loads).

---

#### Interaction tools (unused: all 9)

These tools are designed for testing user journeys — logging in, filling forms,
navigating through a multi-step flow. In a health check, we observe the page
rather than interact with it (except for the consent dialog). In user-journey
testing, these would be your primary tools.

**`click`**
Clicks an element identified by its `uid` from a `take_snapshot`. Simulates a
real mouse click including hover and focus events. Use for: clicking buttons,
links, checkboxes, menu items.

**`hover`**
Moves the mouse over an element without clicking. Use for: triggering tooltips,
dropdown menus, hover animations.

**`drag`**
Drags an element from one position to another. Use for: testing drag-and-drop
interfaces, sliders, sortable lists.

**`fill`**
Types text into a single form field identified by `uid`. Use for: filling in
search boxes, email inputs, password fields.

**`fill_form`**
Fills multiple form fields in one call, given a mapping of field UIDs to values.
More efficient than calling `fill` once per field. Use for: login forms, checkout
forms, registration flows.

**`type_text`**
Types text character by character (simulating real keyboard input, including
triggering `keydown`/`keyup`/`keypress` events). Use when the site uses keyboard
event listeners rather than input value changes (common in rich text editors and
autocomplete components).

**`press_key`**
Presses a specific keyboard key (e.g. `Enter`, `Escape`, `Tab`, `ArrowDown`).
Use for: submitting forms, dismissing modals, navigating dropdowns.

**`upload_file`**
Triggers a file upload on an `<input type="file">` element. Use for: testing
avatar upload, document upload, import flows.

**`handle_dialog`**
Accepts or dismisses `alert()`, `confirm()`, or `prompt()` dialogs. Critical
because these browser dialogs block all further tool calls until dismissed. If
a site uses JavaScript dialogs, you must call this tool to clear them.
> Note: In this health check, CNN used a custom consent modal (not a browser
> dialog), so `handle_dialog` was not needed. We used `evaluate_script` instead.

**`wait_for`** *(used in step 1)*
Already documented above.

---

#### Inspection tools (unused: 2 of 7)

**`get_console_message`**
Retrieves the full details of a single console message by its `msgid`. Use after
`list_console_messages` to get the complete stack trace and all arguments for a
specific message. For example: you see `msgid=141 error: 410 Gone` in the list —
call `get_console_message({ "msgid": 141 })` to get the full URL that returned
410 and the call stack that triggered the request.

**`get_network_request`**
Retrieves the full details of a single network request by its event key. Use
after `list_network_requests` to inspect request headers, response headers, and
response body for a specific request. For example: investigate why an API call
is returning 401, or read the full JSON body of a failed GraphQL mutation.

---

#### Performance tools (unused: 1 of 4)

**`take_memory_snapshot`**
Takes a JavaScript heap snapshot — a complete dump of all JavaScript objects
currently in memory, with their sizes and reference chains. Use for:
- Diagnosing memory leaks (objects that should have been garbage-collected but
  weren't)
- Finding unexpectedly large data structures
- Comparing heap snapshots before and after a user action to see what was
  allocated

This is a specialist tool for memory-related performance problems. A typical
health check does not need it, but it is invaluable when a page's memory usage
grows over time or after repeated actions.

---

#### Audit tools (unused: 2 of 3)

**`emulate`**
Emulates a specific device, network speed, or CPU throttle level. Use this
*before* a performance trace or Lighthouse audit to simulate real-world
conditions:
- `device: "mobile"` — emulate a mobile viewport and touch input
- `network: "Slow4G"` — throttle bandwidth to simulate a slow mobile connection
- `cpuThrottle: 4` — slow the CPU to 25% speed (simulating a mid-range Android
  phone)

Running a performance trace under these conditions reveals how the page performs
for users on slower devices and connections — often dramatically worse than on
a developer's machine.

**`resize_page`**
Resizes the browser viewport to specific dimensions. Use for:
- Testing responsive layout breakpoints
- Ensuring elements are visible at tablet or mobile widths
- Reproducing layout bugs that only appear at specific viewport sizes

---

## Key Lessons from the CNN Health Check

### 1. Real browsers reveal what static analysis cannot
No spider, linter, or static analysis tool could have found the 410 Gone errors,
the sandbox escape warning, the forced reflow from ad verification scripts, or
the 0-second font cache. All of these only exist at runtime in a real browser.

### 2. The accessibility tree is your fastest path to structure
`take_snapshot` gave us a semantic, token-efficient understanding of CNN's page
structure in milliseconds. A screenshot would have required visual processing;
the snapshot gave us a structured document we could reason about directly.

### 3. Performance insights connect data to root causes
Without `performance_analyze_insight`, we would know that CNN's LCP render delay
is 1,537ms but not *why*. The insights tool traced it directly to third-party
script execution — giving us a specific, actionable finding rather than a
raw number.

### 4. `evaluate_script` is your escape hatch
When a page uses non-standard patterns (consent modals in shadow DOM, custom
elements, JavaScript-driven interactions that don't surface in the accessibility
tree), `evaluate_script` lets you reach into the page's own JavaScript context
and do whatever you need.

### 5. The 29 tools are complementary, not redundant
Each tool answers a different question:
- *What does the page look like?* → `take_screenshot`
- *What does the page contain semantically?* → `take_snapshot`
- *What went wrong at runtime?* → `list_console_messages`
- *What did the browser request?* → `list_network_requests`
- *How fast did it load?* → `performance_start_trace` / `performance_stop_trace`
- *What exactly caused the slowness?* → `performance_analyze_insight`
- *How accessible is it?* → `lighthouse_audit`

A complete health check uses tools from every category.

---

## Files Produced

```
demo-health-check/
├── README.md                  ← Index and navigation
├── reference.md               ← Full tool reference (all 29 tools)
├── quickstart.md              ← Hands-on project guide
└── cnn-case-study/
    ├── report.md              ← This health report
    ├── walkthrough.md         ← This document
    └── screenshot.png         ← CNN homepage screenshot
```

---

*Generated using chrome-devtools-mcp — 11 of 29 tools exercised.*
*All data collected from a live browser session on 2026-03-27.*
