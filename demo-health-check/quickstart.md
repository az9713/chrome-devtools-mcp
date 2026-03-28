# Quick-Win Project: Website Health Report
## A Hands-On Introduction to chrome-devtools-mcp

---

## Before We Start — What Is All This?

If you've never heard of MCP, chrome-devtools-mcp, or Chrome DevTools Protocol before, this section is for you. Skip it if you already have context.

### What is an AI coding assistant?

An AI coding assistant is a tool like Claude Code, Cursor, or Gemini CLI — a program you interact with by typing messages, and it helps you write, understand, and debug code. You're probably already using one if you're reading this.

### What is MCP?

MCP stands for **Model Context Protocol**. It's a standard invented by Anthropic that lets AI assistants gain access to external tools — things beyond just reading and writing text. Think of it like a plugin system: instead of the AI only being able to reason about code it can see, MCP lets it call tools like "take a screenshot", "run a search", or in our case, "open a browser and record a performance trace".

When you add an MCP server to your AI client, the AI can now call that server's tools as part of answering your prompts. It happens automatically — you just ask "check the performance of this website" and the AI decides which tools to use.

### What is chrome-devtools-mcp?

`chrome-devtools-mcp` is an MCP server built by Google's Chrome DevTools team. It gives your AI assistant the ability to control and inspect a real Chrome browser. Once it's set up, your AI can:

- Open web pages
- Take screenshots
- Read the page's structure (like an invisible screen reader)
- Collect console errors and network requests
- Run Lighthouse audits (the same tool Chrome DevTools uses for performance/accessibility scoring)
- Record performance traces and get Core Web Vitals (LCP, CLS, INP)

None of this required any coding. You just ask your AI assistant in plain English, and it uses these tools to do the work.

### What are Chrome DevTools?

Chrome DevTools is the panel that opens when you right-click a web page and choose "Inspect". It shows you the page's HTML, CSS, JavaScript errors, network activity, performance data, and more. `chrome-devtools-mcp` gives your AI access to the same underlying data — without you having to manually open DevTools yourself.

### What is Lighthouse?

Lighthouse is an automated auditing tool built into Chrome that scores a webpage on:
- **Performance** — how fast it loads
- **Accessibility** — how usable it is for people with disabilities
- **Best Practices** — modern web development standards
- **SEO** — search engine optimization

You've probably seen the colored score circles if you've ever opened Chrome DevTools → Lighthouse tab. `chrome-devtools-mcp` lets your AI run these audits and read the results.

### What are Core Web Vitals?

Core Web Vitals are Google's official measurements for how good a user's experience is on a webpage:

- **LCP (Largest Contentful Paint)** — how long until the main content appears. Under 2.5 seconds is "Good".
- **CLS (Cumulative Layout Shift)** — how much the layout jumps around while loading. Under 0.1 is "Good".
- **INP (Interaction to Next Paint)** — how quickly the page responds to clicks/taps. Under 200ms is "Good".

These numbers directly affect Google search rankings and user satisfaction.

---

## What You'll Build

By the end of this project, you'll have run a complete website health check using your AI assistant. The AI will use `chrome-devtools-mcp` tools to:

1. Open a website in Chrome
2. Understand the page structure
3. Take a screenshot
4. Collect any console errors and network issues
5. Run a Lighthouse audit
6. Record a performance trace and extract Core Web Vitals
7. Combine all of this into a health report you can save and share

**Time required:** ~15-20 minutes
**Coding required:** None — you'll just type prompts
**What you'll end up with:**

```
health-report/
├── report.md           ← Full health report in Markdown
├── lighthouse.html     ← Lighthouse report you can open in your browser
└── screenshot.png      ← Screenshot of the page
```

---

## Prerequisites

### 1. Node.js (v20.19 or newer)

Node.js is the JavaScript runtime that `chrome-devtools-mcp` runs on. You can check if you already have it:

```bash
node --version
```

If you see something like `v20.19.0` or higher, you're good. If the command isn't found, download Node.js from https://nodejs.org — get the "LTS" version.

### 2. npm

npm comes bundled with Node.js. Verify:

```bash
npm --version
```

Any version should work.

### 3. Google Chrome (current stable version)

Download from https://www.google.com/chrome if you don't have it. Other Chromium-based browsers (Edge, Brave, Arc) are **not** guaranteed to work — use Chrome.

### 4. An MCP-compatible AI client

You need an AI assistant that supports MCP. This guide was written with **Claude Code** in mind, but the same approach works with Gemini CLI, Cursor, Windsurf, and others. The setup step below is for Claude Code.

If you're using a different client, the configuration step will differ slightly — but the prompts in Steps 1–6 are identical regardless of which client you use.

---

## Setup: Adding chrome-devtools-mcp to Claude Code

This is a one-time setup. After this, every Claude Code session will have access to the browser tools.

### Option A: Using the CLI (easiest)

Open your terminal and run:

```bash
claude mcp add chrome-devtools --scope user npx chrome-devtools-mcp@latest
```

This registers the MCP server globally for your user account. You only need to do this once.

**Where Claude Code stores MCP config:**

| Scope | File |
|-------|------|
| User (all projects) | `~/.claude.json` (Windows: `C:\Users\<you>\.claude.json`) |
| Project only | `.claude/mcp.json` inside the project folder |

`claude mcp add --scope user` writes to `~/.claude.json` automatically.

### Option B: Manual configuration

If the CLI command doesn't work, you can add it manually. Open (or create) the Claude Code user config file. On Windows it's at:

```
C:\Users\<your-username>\.claude.json
```

On Mac/Linux:
```
~/.claude.json
```

> **Note:** This is different from `claude_desktop_config.json` which is used by the Claude Desktop GUI app. Claude Code CLI uses `~/.claude.json` (a file directly in your home directory, not inside a folder).

Add this JSON (or merge it into the `mcpServers` section if that key already exists):

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

### Verify the setup

**Restart Claude Code** after making the change.

Then ask Claude:

> Do you have access to a chrome-devtools MCP server? List the available browser tools.

Claude should respond by listing tools like `navigate_page`, `take_snapshot`, `take_screenshot`, etc. If it says it doesn't have those tools, the MCP server isn't connected — check the troubleshooting section at the bottom of this guide.

---

## Choosing Your Target Website

You need a URL to run the health check on. Pick one:

| Option | Pros | Cons |
|--------|------|------|
| `https://developers.chrome.com` | Safe public site, always available, interesting results | Not your site, can't fix the issues |
| Your own project | Most relevant findings, you can actually fix issues | Must be running (localhost or deployed) |
| Any public website you use | Interesting to see results for something familiar | Can't fix issues |

For your first run, `https://developers.chrome.com` is a great choice — it's a real production site with real performance characteristics, and it's always available.

> **Note:** Do not run this on sites you don't own without permission for anything beyond read-only inspection. The tools used here are read-only (no form submissions, no clicks), so observing public sites is fine.

---

## The Project: 6 Prompts

Copy each prompt below into your AI assistant. Read the "What's happening" section after each one to understand what the tools are doing.

### Step 1 — Navigate and Understand the Page

**Paste this prompt:**

```
Navigate to https://developers.chrome.com (or replace with your chosen URL).

Then take a text snapshot of the page. Based on the snapshot:
1. Describe the overall page structure (header, main content, navigation, footer)
2. List the main interactive elements (buttons, links, forms)
3. Note any accessibility issues you can detect from the snapshot alone
4. Tell me what the page appears to be about

Save your analysis to ./health-report/step1-snapshot-analysis.md
```

**What's happening:**

The AI uses two tools here:
- `navigate_page` — opens the URL in Chrome (Chrome launches automatically if it's not running yet, using a managed profile in `~/.cache/chrome-devtools-mcp/chrome-profile`)
- `take_snapshot` — reads the page's **accessibility tree**, not the raw HTML. The accessibility tree is what screen readers use — it's a structured, semantic representation of the page that lists every button, heading, link, image, and form field with their labels and roles. It's token-efficient (much smaller than full HTML) and easier for the AI to reason about.

Each element in the snapshot gets a stable **UID** (like `3_42`) that persists across snapshots. If you later want the AI to click a specific button, it uses that UID. This is how the AI can reliably interact with specific elements across multiple tool calls.

**What to expect:** Claude will describe the page layout in plain English and flag any obvious accessibility issues (missing image descriptions, unlabeled buttons, etc.).

---

### Step 2 — Take a Screenshot

**Paste this prompt:**

```
Take a screenshot of the current page and save it to ./health-report/screenshot.png
```

**What's happening:**

`take_screenshot` captures what the page actually looks like visually in the browser Chrome opened.

**What to expect:** Claude will take the screenshot and confirm the file was saved. Open `health-report/screenshot.png` to see what Chrome is looking at. This confirms the browser is working and showing the right page.

**Why both snapshot and screenshot?** They serve different purposes:
- The **snapshot** (Step 1) is structured text — the AI can reason about it, count elements, find specific components
- The **screenshot** is visual — useful for you to see the page, and for the AI to notice visual layout issues

---

### Step 3 — Check for Console Errors and Network Issues

**Paste this prompt:**

```
List all console messages for the current page. Then list all network requests.

Based on what you find:
1. Are there any JavaScript errors? If so, what are they?
2. Are there any failed network requests (status 4xx or 5xx)?
3. Are there any console warnings that suggest potential problems?
4. What are the largest resources loaded by the page? (images, scripts, etc.)

Save your findings to ./health-report/step3-console-network.md
```

**What's happening:**

Two tools run here:
- `list_console_messages` — retrieves all messages logged to the browser console since the page loaded. This includes `console.log`, `console.warn`, `console.error`, and uncaught JavaScript exceptions. Source-mapped stack traces are included where available, so you see the original TypeScript/JSX filename and line number rather than minified code.
- `list_network_requests` — retrieves every HTTP request the page made, including resource type (script, stylesheet, image, XHR/fetch), HTTP status code, size, and timing. This is the same information you'd see in Chrome DevTools → Network tab.

**What to expect:** For most production sites, you'll see a mix of 200 OK responses, possibly some 3xx redirects, and maybe a few 4xx errors for missing resources. JavaScript errors are common even on polished sites. The AI will categorize these and explain which ones matter.

---

### Step 4 — Run a Lighthouse Audit

**Paste this prompt:**

```
Run a Lighthouse audit on the current page in navigation mode on desktop.
Save the HTML report to ./health-report/lighthouse.html

Tell me:
1. The scores for Accessibility, SEO, and Best Practices (0-100)
2. The most critical accessibility issues found
3. The most critical SEO issues found
4. Any Best Practices violations

Save a summary to ./health-report/step4-lighthouse.md
```

**What's happening:**

`lighthouse_audit` runs Google Lighthouse directly against the current Chrome session. Lighthouse is the same engine that powers the "Lighthouse" tab in Chrome DevTools and Google's PageSpeed Insights tool.

It audits the page against hundreds of checks and produces scores in each category. For each failing audit, it explains what's wrong and how to fix it.

**Modes:**
- `navigation` mode — Lighthouse reloads the page from scratch and measures it during load (best for performance and load-related issues)
- `snapshot` mode — Lighthouse analyzes the current page state without reloading (faster, useful for checking the current DOM)

We're using `navigation` mode to get a fresh, clean measurement.

**What to expect:** Claude will report scores like "Accessibility: 87, SEO: 92, Best Practices: 95" and then list specific failures. Common accessibility issues include missing `alt` text on images, insufficient color contrast, and unlabeled form fields. SEO issues often include missing meta descriptions or non-descriptive link text.

The `lighthouse.html` file is the full Lighthouse report — open it in your browser for the beautiful color-coded version with detailed explanations for every issue.

---

### Step 5 — Record a Performance Trace

**Paste this prompt:**

```
Record a performance trace of the current page loading. Use the default settings (reload: true, autoStop: true).

After the trace, tell me:
1. The LCP (Largest Contentful Paint) time and rating (Good/Needs Improvement/Poor)
2. The CLS (Cumulative Layout Shift) score and rating
3. The INP (Interaction to Next Paint) if available
4. What element caused the LCP (the largest visible element)
5. The main performance bottleneck — what's making the page slow?
6. The top 3 concrete things I could do to improve performance

Save your analysis to ./health-report/step5-performance.md
```

**What's happening:**

This is the most powerful and unique capability of chrome-devtools-mcp.

- `performance_start_trace` — starts a Chrome DevTools performance trace recording. This captures hundreds of trace events: every script execution, layout, paint, network request, and rendering event that Chrome performs while loading the page. The trace uses the same categories as Chrome DevTools → Performance tab.
- Because `reload: true` is set, it navigates to `about:blank` first to clear cached state, then reloads the original URL while recording.
- Because `autoStop: true` is set, recording stops automatically when the page finishes loading.
- The raw trace is then analyzed by the same trace-parsing code used in Chrome DevTools itself, extracting semantic insights.

**What the AI receives:** Not raw trace JSON (which would be megabytes of data). Instead, it receives a semantic summary: "LCP was 2.1s, classified as 'Good'. The LCP element is the hero image. Resource load delay contributes 0.8s of that. No layout shifts detected."

This is `chrome-devtools-mcp`'s token optimization at work — the AI gets the answer, not the raw data.

**What to expect:** You'll get precise millisecond timings, ratings against Google's thresholds, and specific actionable recommendations like "the hero image is not lazy-loaded" or "render-blocking JavaScript is delaying the first contentful paint by 400ms".

---

### Step 6 — Generate the Full Health Report

**Paste this prompt:**

```
You've now gathered everything from steps 1-5:
- Page structure and accessibility snapshot (step1-snapshot-analysis.md)
- A screenshot (screenshot.png)
- Console errors and network issues (step3-console-network.md)
- Lighthouse scores and audit findings (step4-lighthouse.md)
- Performance trace and Core Web Vitals (step5-performance.md)

Combine all of this into a comprehensive Website Health Report saved to ./health-report/report.md

The report should include:
- Executive Summary (2-3 sentences: is this site healthy overall?)
- A scorecard table: Accessibility score, SEO score, Best Practices score, LCP rating, CLS rating
- Section: Console & Network Health (key errors and failed requests)
- Section: Accessibility Findings (top issues with code examples if relevant)
- Section: SEO Findings
- Section: Performance Analysis (Core Web Vitals with pass/fail, main bottleneck, waterfall summary)
- Section: Top 5 Recommendations (ordered by impact, specific and actionable)

Use markdown formatting. Include the screenshot with ![Page Screenshot](./screenshot.png)
```

**What's happening:**

No new browser tools are called here. The AI synthesizes the analysis it already collected across the previous five steps into a single coherent document.

**What to expect:** A well-structured markdown report you can open in any markdown viewer, share with teammates, or use as a baseline to track improvements over time. The `report.md` will reference the screenshot and link to the `lighthouse.html` report.

---

## What You Should Now Understand

After completing these 6 steps, you've touched the following `chrome-devtools-mcp` tools:

| Tool | Category | What it does |
|------|----------|--------------|
| `navigate_page` | Navigation | Opens a URL in Chrome |
| `take_snapshot` | Debugging | Reads the accessibility tree — semantic page structure |
| `take_screenshot` | Debugging | Captures a visual screenshot |
| `list_console_messages` | Debugging | Gets JS errors, warnings, logs |
| `list_network_requests` | Network | Gets all HTTP requests and their status |
| `lighthouse_audit` | Debugging | Runs Lighthouse accessibility/SEO/best-practices audit |
| `performance_start_trace` | Performance | Records Chrome DevTools performance trace, extracts Core Web Vitals |

That's 7 of the 29 available tools — and they cover the two most impactful feature areas:
- **Debugging** (snapshot + console + network + Lighthouse)
- **Performance analysis** (trace + Core Web Vitals)

---

## Bonus Challenges

Once you've completed the basic project, try these to go deeper:

### Challenge 1: Mobile Emulation

Before Step 5, add this prompt:

```
Emulate a Slow 4G network connection and a mobile viewport (375x812 pixels, isMobile: true).
Then re-run the performance trace.
Compare the LCP and other Core Web Vitals to the desktop results.
```

The `emulate` tool throttles the network and resizes the browser to simulate a mobile device. You'll likely see significantly worse performance numbers — which is exactly what real mobile users experience.

### Challenge 2: Fix and Verify

If the report found a fixable issue (a missing `alt` tag, a missing meta description, etc.):

1. Fix it in your code
2. Reload your local development server
3. Ask the AI:
   ```
   Navigate to http://localhost:3000 (your local URL).
   Run a Lighthouse audit focused on accessibility.
   Has the missing alt text issue been fixed?
   ```

This shows the full debug → fix → verify loop that `chrome-devtools-mcp` enables.

### Challenge 3: Compare Two Sites

```
Run the full health check on both https://site-a.com and https://site-b.com.
Produce a side-by-side comparison table of their Core Web Vitals,
Lighthouse scores, and top issues.
```

This uses `new_page` to open a second tab and `select_page` to switch between them — two more tools in the navigation category.

---

## Troubleshooting

### "I don't have access to chrome-devtools tools"

The MCP server isn't connected. Try:
1. Restart your AI client (Claude Code / Cursor / etc.)
2. Check your config file is valid JSON (`jsonlint.com` can validate it)
3. Run `claude mcp list` in your terminal to see if `chrome-devtools` appears

### "Chrome won't launch" or "Failed to launch browser"

- Make sure Google Chrome is installed (not just Chromium or Edge)
- On Windows, try specifying the path explicitly in your config:
  ```json
  "args": ["-y", "chrome-devtools-mcp@latest", "--executable-path=C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe"]
  ```
- Check if another process is holding the Chrome profile open: look for a running Chrome instance that was launched by a previous MCP session

### "The health-report directory doesn't exist"

The AI should create it automatically when it writes the first file. If it doesn't, ask:
```
Create a directory called health-report in the current working directory, then continue.
```

### The performance trace takes a long time

Performance traces can take 30-60 seconds — Chrome is reloading the page, recording all events, and then processing the trace data. This is normal. Don't interrupt it.

### Lighthouse audit fails with a timeout

Some pages take too long to load for Lighthouse. Try:
```
Run a Lighthouse audit in snapshot mode instead of navigation mode.
```

Snapshot mode analyzes the current page state instead of doing a fresh load, so it's faster and more forgiving.

### "The screenshot is blank / shows about:blank"

This can happen if the page navigation didn't complete before the screenshot was taken. Ask:
```
Navigate to [URL] again and wait for the page to fully load, then take a screenshot.
```

---

## What's Next

Now that you've completed the basic project, here's where to go next:

**Learn the input/automation tools:** Try asking the AI to click buttons, fill forms, and navigate through a multi-step flow. The `click`, `fill`, `type_text`, and `handle_dialog` tools enable full browser automation.

**Try the auto-connect mode (Chrome 144+):** Instead of using a managed Chrome instance, connect to your real running Chrome session. This lets the AI work on pages where you're already logged in. Enable it at `chrome://inspect/#remote-debugging` and add `--auto-connect` to your MCP config.

**Run the accessibility debugging skill:** If your AI client supports skills (Claude Code does), the `a11y-debugging` skill provides a structured workflow for hunting down and fixing specific accessibility issues using `chrome-devtools-mcp`.

**Read the full documentation:** `reference.md` in this folder covers all 29 tools, all configuration options, the daemon/CLI, and the full architecture.
