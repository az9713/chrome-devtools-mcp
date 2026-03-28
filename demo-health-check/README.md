# demo-health-check

Our own documentation for learning and using `chrome-devtools-mcp`.
These files are independent of the original repo's `docs/` folder.

---

## Contents

### [`reference.md`](reference.md)
Complete reference for all 29 chrome-devtools-mcp tools, the architecture, connection modes, daemon/CLI, and configuration options. Start here if you want to understand the full surface area of the tool.

### [`quickstart.md`](quickstart.md)
A hands-on guided project: produce a Website Health Report for any URL using 7 of the 29 tools, in ~15 minutes, with 6 copy-paste prompts. Includes setup instructions, prerequisites, troubleshooting, and bonus challenges.

### [`cnn-case-study/`](cnn-case-study/)
A real-world run of the health check project against CNN.com.

| File | What it is |
|------|-----------|
| [`report.md`](cnn-case-study/report.md) | The health report — findings, Core Web Vitals, third-party analysis, forced reflow data, cache audit, recommendations |
| [`walkthrough.md`](cnn-case-study/walkthrough.md) | Step-by-step annotated walkthrough of every tool used, every input/output, and every obstacle encountered — the full story behind the report |
| [`screenshot.png`](cnn-case-study/screenshot.png) | CNN homepage screenshot taken during the session |

---

## Trigger prompt

Copy this into Claude Code (with chrome-devtools-mcp configured) to run this demo yourself:

> "Using chrome-devtools-mcp, run a full website health check on https://www.cnn.com. Open the page in a new tab, dismiss any consent or cookie dialogs that appear, then: take a screenshot, read the accessibility tree, start a performance trace and reload the page, stop the trace and analyze Core Web Vitals (LCP, CLS, INP), list all network requests filtering for third-party domains and count unique vendor domains, check for console errors, and attempt a Lighthouse audit. Write a health report covering: page structure, performance scores, third-party vendor count, any forced reflow or layout thrashing issues, cache headers, and your top 5 recommendations."

---

## Suggested Reading Order

1. `quickstart.md` — understand what you'll build and get set up
2. `cnn-case-study/report.md` — see a real output
3. `cnn-case-study/walkthrough.md` — understand exactly how it was produced
4. `reference.md` — go deep on any tool or concept
