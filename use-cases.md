# chrome-devtools-mcp — AI Agentic Use Cases

A catalogue of workflows where an AI agent with access to `chrome-devtools-mcp` can do genuinely useful work. Two demos are already documented in this repo; the list below covers what else is possible across every domain.

---

## How to read this

Each use case notes:
- **What the agent does** — the concrete task
- **Key tools** — which of the 29 tools drive it
- **Why it beats alternatives** — what makes the CDP approach better than a plain web scraper or a Lighthouse run

---

## 1. Quality Assurance & Testing

### 1.1 Regression snapshot testing
The agent opens every page in a sitemap, takes a full-page screenshot, and diffs it against a stored baseline. Any pixel change surfaces as a finding.

**Key tools:** `navigate_page`, `take_screenshot` (fullPage), `list_network_requests`
**Why CDP:** Screenshots include fonts, custom CSS, lazy-loaded images — things headless screenshot tools often miss.

### 1.2 End-to-end form validation
The agent fills every form field on a page with boundary values (empty, max length, special characters, SQL fragments) and records which inputs produce error states vs. silent failures.

**Key tools:** `fill_form`, `fill`, `type_text`, `take_snapshot`, `list_console_messages`
**Why CDP:** The a11y snapshot tells the agent whether an error message appeared without needing to parse pixel positions.

### 1.3 Cross-device responsive QA
The agent loads a page at 10 different viewport sizes — phone portrait, phone landscape, tablet, wide desktop — and compares the a11y tree and screenshots at each. Detects layout reflow, hidden nav items, and overlapping elements.

**Key tools:** `emulate`, `resize_page`, `take_screenshot`, `take_snapshot`
**Why CDP:** Single browser session, no fleet of devices needed. DPR and touch emulation are accurate.

### 1.4 Broken link and 404 detector
The agent crawls a site, follows every `<a>` tag, and reports all links that result in a 4xx or 5xx HTTP status.

**Key tools:** `take_snapshot`, `navigate_page`, `list_network_requests`
**Why CDP:** Catches links that redirect through JS before issuing the final request — pure HTML scrapers miss these.

### 1.5 Dialog and modal interaction testing
The agent triggers every modal on a page (cookie banners, age gates, newsletter popups, confirm dialogs) and verifies each can be dismissed without leaving the page in a broken state.

**Key tools:** `click`, `handle_dialog`, `take_snapshot`
**Why CDP:** `handle_dialog` intercepts native browser `alert()`/`confirm()`/`prompt()` — impossible to automate from outside the browser.

### 1.6 Authentication flow testing
The agent runs through login, logout, session expiry, wrong password, locked account, and "forgot password" flows. Records each state via snapshot and screenshot.

**Key tools:** `fill_form`, `click`, `navigate_page`, `take_snapshot`, `list_console_messages`
**Why CDP:** Can inspect network requests to verify tokens are being sent correctly alongside verifying the UI state.

---

## 2. Performance Engineering

### 2.1 Core Web Vitals CI gate
Before a deploy, the agent runs a performance trace, extracts LCP, CLS, and INP, and fails the build if any metric regresses beyond a threshold. Runs against staging with real network conditions.

**Key tools:** `performance_start_trace`, `performance_stop_trace`, `performance_analyze_insight`
**Why CDP:** Measures real browser rendering metrics — not synthetic estimates. Catches regressions that Lighthouse misses in practice.

### 2.2 Third-party script impact audit
The agent loads a page twice: once normally, once with all third-party domains blocked via request interception. It records the performance delta to quantify the cost of each vendor (analytics, ads, chat widgets, etc.).

**Key tools:** `performance_start_trace`, `performance_analyze_insight`, `list_network_requests`, `get_network_request`
**Why CDP:** Can pinpoint which exact JS file from which domain caused the longest task on the main thread.

### 2.3 Memory leak hunter
The agent navigates through an SPA route sequence 10 times, taking a heap snapshot before and after. It compares the two snapshots and surfaces any object category whose count grew monotonically.

**Key tools:** `take_memory_snapshot`, `navigate_page`
**Why CDP:** V8 heap snapshots are the gold standard for JS memory analysis. No other tool gives this level of detail without native DevTools.

### 2.4 Slow-network experience audit
The agent throttles the connection to "Slow 3G", loads a page, and records which resources block rendering, what the LCP element is, and how long the page takes to become interactive.

**Key tools:** `emulate` (networkConditions), `performance_start_trace`, `performance_analyze_insight`, `take_screenshot`
**Why CDP:** Tests the real experience on slow connections without needing a physical device on a slow network.

### 2.5 CPU throttle regression test
The agent emulates a mid-range Android device by applying 4× CPU throttling, then runs a user interaction (scroll, accordion open, modal toggle) and measures how long it takes. Flags interactions over 200 ms.

**Key tools:** `emulate` (cpuThrottlingRate), `performance_start_trace`, `click`, `performance_analyze_insight`
**Why CDP:** Directly reproduces the experience of a low-end device without owning one.

---

## 3. Accessibility & Compliance

### 3.1 WCAG accessibility audit pipeline
The agent runs a Lighthouse accessibility audit on every page of a site and aggregates the results into a spreadsheet-ready report: missing alt text, insufficient colour contrast, unlabelled form fields, missing landmark roles.

**Key tools:** `lighthouse_audit`, `navigate_page`, `take_snapshot`
**Why CDP:** Lighthouse's accessibility engine runs inside Chrome — it sees the fully-rendered DOM including dynamically injected content.

### 3.2 Keyboard-only navigation test
The agent navigates a page using only `Tab`, `Shift+Tab`, `Enter`, and `Space`. It records the focus order via the a11y tree and flags any interactive element that is unreachable by keyboard or that loses visible focus styling.

**Key tools:** `press_key`, `take_snapshot`, `take_screenshot`
**Why CDP:** The a11y tree shows `focusable` and `focused` state directly — no pixel analysis required.

### 3.3 Screen reader simulation
The agent traverses the accessibility tree of a page in the same order a screen reader would (depth-first, ARIA roles respected) and narrates the experience. Surfaces missing labels, wrong heading hierarchy, and decorative images that lack `role="presentation"`.

**Key tools:** `take_snapshot` (verbose mode), `navigate_page`
**Why CDP:** The full a11y tree exposes role, label, state, and description for every node — exactly what a screen reader sees.

### 3.4 Cookie consent compliance checker
The agent loads a page without any cookies set, records which third-party requests fire before the user interacts with the consent banner, and flags any that fire before consent is given.

**Key tools:** `navigate_page`, `list_network_requests`, `take_snapshot`, `click`
**Why CDP:** Captures the precise moment each network request fires relative to user interaction — essential for GDPR pre-consent evidence.

---

## 4. SEO & Content

### 4.1 SEO technical audit
The agent runs a Lighthouse SEO audit, then augments it with its own checks: missing canonical tags, duplicate `<title>` elements across pages, missing `hreflang`, non-indexable status codes.

**Key tools:** `lighthouse_audit`, `navigate_page`, `evaluate_script`
**Why CDP:** Sees the page after JS has executed — catches tags injected by React/Next/Nuxt that a static crawler would miss.

### 4.2 Structured data validator
The agent extracts all JSON-LD and microdata from a page, validates it against schema.org specs, and reports missing required fields (e.g. a `Product` schema missing `price` or `availability`).

**Key tools:** `evaluate_script`, `navigate_page`
**Why CDP:** Runs inside the browser — catches structured data injected after page load by tag managers.

### 4.3 Competitor content tracker
The agent visits a set of competitor pages on a schedule, extracts headings, product descriptions, and pricing, and produces a diff report showing what changed since the last run.

**Key tools:** `navigate_page`, `take_snapshot`, `evaluate_script`
**Why CDP:** Renders JS-heavy pages correctly, so it doesn't miss content that a plain HTTP scraper would return as empty.

### 4.4 Meta tag completeness checker
The agent checks every page in a sitemap for Open Graph tags, Twitter Card tags, canonical URLs, and meta descriptions. Produces a pass/fail table.

**Key tools:** `navigate_page`, `evaluate_script`
**Why CDP:** Works on SPAs where meta tags are injected by client-side frameworks.

---

## 5. E-commerce & Retail

### 5.1 Price monitoring agent
The agent visits a list of product pages daily, extracts the current price and availability, and alerts when a price drops or an out-of-stock item comes back.

**Key tools:** `navigate_page`, `evaluate_script`, `take_screenshot`
**Why CDP:** Handles sites that require JS rendering to show the real price (after personalisation, geolocation, or login).

### 5.2 Checkout funnel QA
The agent runs through a complete purchase flow on staging before every deploy — add to cart, apply discount code, fill shipping, confirm order — and verifies each step produces the correct UI state and network response.

**Key tools:** `fill_form`, `click`, `evaluate_script`, `list_network_requests`, `get_network_request`
**Why CDP:** Inspects the actual API calls made during checkout to verify the right payload is being sent, not just that the UI looks correct.

### 5.3 Discount code validator
The agent tries a list of discount codes on a checkout page and records which are valid, which are expired, and what each applies to. Useful for auditing a promo campaign before launch.

**Key tools:** `fill`, `click`, `take_snapshot`, `evaluate_script`

### 5.4 Product catalogue crawler
The agent pages through an entire product catalogue, extracts title, price, SKU, image URL, and description for every item, and exports a structured JSON or CSV.

**Key tools:** `navigate_page`, `take_snapshot`, `evaluate_script`, `click`
**Why CDP:** Handles infinite scroll, "load more" buttons, and session-gated catalogues.

### 5.5 Flash sale monitor
The agent polls a product page every 60 seconds, detects when the price drops or a "Limited time" badge appears, and immediately screenshots the offer and triggers a notification.

**Key tools:** `navigate_page`, `evaluate_script`, `take_screenshot`

---

## 6. Research & Information Gathering

### 6.1 Multi-source research synthesiser
The agent opens 10–20 tabs simultaneously (one per source), reads the accessibility tree of each, and synthesises the information into a structured briefing — without the user having to read each page manually.

**Key tools:** `new_page`, `list_pages`, `select_page`, `take_snapshot`, `close_page`
**Why CDP:** Multi-tab management lets the agent hold a "reading list" in parallel and switch between sources without losing state.

### 6.2 Academic literature review
The agent searches a database (PubMed, Google Scholar, arXiv), opens each result, extracts abstract and metadata, and produces an annotated bibliography sorted by relevance.

**Key tools:** `navigate_page`, `fill`, `press_key`, `take_snapshot`, `evaluate_script`

### 6.3 Job market research
The agent searches multiple job boards for a given role + location, extracts job titles, companies, salaries, and required skills, and produces a salary benchmark and skills frequency report.

**Key tools:** `navigate_page`, `fill_form`, `take_snapshot`, `evaluate_script`

### 6.4 Real estate listing aggregator
The agent visits Zillow, Redfin, Realtor.com, and local agency sites, extracts listings matching given criteria, and produces a side-by-side comparison table with price-per-sqft calculated.

**Key tools:** `navigate_page`, `fill_form`, `take_snapshot`, `evaluate_script`, `list_pages`, `select_page`

### 6.5 Flight and hotel price matrix
Given a set of travel dates and destinations, the agent visits booking sites, extracts prices for each combination, and returns the cheapest option for each route.

**Key tools:** `navigate_page`, `fill_form`, `click`, `take_snapshot`, `evaluate_script`

---

## 7. Developer Tooling & DevOps

### 7.1 Storybook component screenshot library
The agent opens every story in a Storybook instance, takes a screenshot of each, and builds a visual component library — updated automatically on every CI run.

**Key tools:** `navigate_page`, `take_snapshot`, `take_screenshot`
**Why CDP:** Storybook renders in a real browser — components look exactly as they do in production.

### 7.2 API response inspector
The agent navigates an authenticated web app, performs actions that trigger API calls, and inspects the request/response pairs. Useful for documenting undocumented internal APIs.

**Key tools:** `navigate_page`, `click`, `list_network_requests`, `get_network_request`
**Why CDP:** Captures the full request body, response body, and all headers — including auth tokens — without needing API docs.

### 7.3 Console error triage
The agent loads a list of URLs, collects all console errors and warnings from each, de-duplicates them, groups by error message, and produces a prioritised fix list with page URLs attached.

**Key tools:** `navigate_page`, `list_console_messages`, `get_console_message`
**Why CDP:** Captures runtime JS errors that only appear after user interaction — not just on initial load.

### 7.4 Feature flag verifier
The agent loads a page in different conditions (logged in vs. out, different user segments, different geographies via geolocation emulation) and confirms which UI elements appear in each condition.

**Key tools:** `navigate_page`, `emulate` (geolocation), `take_snapshot`, `evaluate_script`

### 7.5 Localisation QA
The agent loads a page with different `Accept-Language` headers and locale settings, and checks for untranslated strings, layout overflow caused by longer translated text, and missing locale-specific date/currency formats.

**Key tools:** `emulate` (userAgent), `navigate_page`, `take_snapshot`, `take_screenshot`

### 7.6 Dark mode visual audit
The agent takes screenshots of every page in both light and dark mode, and flags any element where the text-to-background contrast ratio drops below WCAG AA threshold.

**Key tools:** `emulate` (colorScheme), `take_screenshot`, `navigate_page`

---

## 8. Content Creation & Documentation

### 8.1 Automated tutorial screenshot generator
Given a user flow (e.g. "how to set up a project in Linear"), the agent runs through each step and saves a numbered screenshot at every click. Produces a ready-to-use image sequence for a how-to article.

**Key tools:** `navigate_page`, `fill_form`, `click`, `take_screenshot`
**Why CDP:** Screenshots are taken at the exact right moment — after state changes settle — not mid-transition.

### 8.2 Release notes screenshot pack
Before a product release, the agent visits every changed page/feature in a staging environment and captures before/after screenshots. PMs get a visual diff without opening the browser themselves.

**Key tools:** `navigate_page`, `take_screenshot`, `emulate`

### 8.3 Website-to-PDF report generator
The agent navigates a multi-page web report (dashboards, analytics tools, CMS previews), takes full-page screenshots of each section, and assembles them into a PDF.

**Key tools:** `navigate_page`, `take_screenshot` (fullPage), `evaluate_script`

### 8.4 Design-to-implementation comparison
The agent takes a screenshot of the live implementation, a designer shares a Figma export, and the agent overlays them to identify spacing, colour, and typography discrepancies.

**Key tools:** `take_screenshot`, `navigate_page`, `emulate`

---

## 9. Security & Compliance

### 9.1 Exposed secrets scanner
The agent loads a set of pages and inspects all inline JS, network request payloads, and console output for patterns that look like API keys, tokens, or credentials (`sk_live_`, `AKIA`, etc.).

**Key tools:** `navigate_page`, `evaluate_script`, `list_network_requests`, `get_network_request`, `list_console_messages`
**Why CDP:** Sees the fully evaluated JS, including strings that are only assembled at runtime.

### 9.2 Mixed content detector
The agent loads HTTPS pages and flags any resource (image, script, iframe) loaded over HTTP — a common cause of browser security warnings.

**Key tools:** `navigate_page`, `list_network_requests`
**Why CDP:** Captures every sub-resource request including those triggered by third-party scripts.

### 9.3 CSP violation reporter
The agent loads a page and reads any `[Security]` console messages emitted by the browser's Content Security Policy engine. Maps each violation to the script or resource that caused it.

**Key tools:** `navigate_page`, `list_console_messages`, `get_console_message`
**Why CDP:** CSP violations are emitted as native browser console events — only visible from inside the browser.

### 9.4 Analytics event verifier
The agent performs a user action (page view, button click, form submit) and inspects the outgoing network requests to confirm the correct analytics events fired with the correct properties. Catches tracking regressions before they make it to production.

**Key tools:** `click`, `fill_form`, `list_network_requests`, `get_network_request`
**Why CDP:** Can filter for XHR/fetch requests to known analytics endpoints (`gtm`, `analytics`, `segment`, `mixpanel`) and inspect the exact payload.

---

## 10. Business Operations & Automation

### 10.1 SaaS onboarding automation
The agent creates a free trial account on a SaaS tool, completes the onboarding wizard, configures a workspace, and produces a summary of what the product actually does — without a human spending an hour clicking through signup flows.

**Key tools:** `fill_form`, `click`, `handle_dialog`, `take_snapshot`, `take_screenshot`

### 10.2 Invoice and form filing automation
The agent logs into a government or supplier portal, fills in a recurring form (tax filing, expense report, supplier invoice), and submits it — handling CAPTCHAs and session timeouts gracefully.

**Key tools:** `fill_form`, `fill`, `click`, `press_key`, `upload_file`, `handle_dialog`

### 10.3 Competitor feature matrix builder
The agent visits the pricing and features pages of 20 competitors, extracts the feature list for each tier, and builds a comparison matrix. Updated monthly without human effort.

**Key tools:** `navigate_page`, `take_snapshot`, `evaluate_script`, `new_page`, `list_pages`, `select_page`

### 10.4 Job application assistant
The agent fills in job application forms on company career portals: name, contact details, work history, cover letter, CV upload — using a stored candidate profile. Logs each application with a screenshot of the confirmation page.

**Key tools:** `fill_form`, `upload_file`, `click`, `take_screenshot`

### 10.5 Social proof aggregator
The agent visits review platforms (G2, Capterra, Trustpilot, Product Hunt), extracts recent reviews for a given product, and produces a sentiment summary with notable quotes.

**Key tools:** `navigate_page`, `take_snapshot`, `evaluate_script`

### 10.6 Regulatory filing monitor
The agent checks government filing portals daily for new submissions from a list of companies (SEC EDGAR, Companies House, patent offices), and surfaces any new filings with a summary.

**Key tools:** `navigate_page`, `fill_form`, `take_snapshot`, `evaluate_script`

---

## 11. Personal Productivity

### 11.1 Daily briefing builder
Each morning the agent opens a custom list of news sites, newsletters, and dashboards, extracts the key content from each, and assembles a one-page digest — delivered before you open your laptop.

**Key tools:** `new_page`, `navigate_page`, `take_snapshot`, `evaluate_script`, `close_page`

### 11.2 Grocery order automator
The agent logs into a grocery delivery site, adds a fixed weekly list of items to the basket, applies any available discount codes, and completes the order — saving 15 minutes every week.

**Key tools:** `fill`, `click`, `evaluate_script`, `handle_dialog`, `take_screenshot`

### 11.3 Portfolio tracker
The agent visits brokerage and crypto dashboards (which don't have APIs, or whose APIs require paid access), extracts current holdings and values, and aggregates them into a unified net-worth summary.

**Key tools:** `navigate_page`, `fill_form`, `evaluate_script`, `take_snapshot`
**Why CDP:** Works on sites behind login and with client-side-rendered data that no public API exposes.

### 11.4 Meeting prep researcher
Given a calendar invite with company names and LinkedIn URLs, the agent visits each company's website and the attendees' LinkedIn profiles, extracts key facts, and produces a one-paragraph brief per person/company.

**Key tools:** `navigate_page`, `take_snapshot`, `evaluate_script`

---

## Summary: Tool–Use Case Matrix

| Tool category | Best-fit use cases |
|---------------|--------------------|
| **Navigation** (`new_page`, `navigate_page`, `list_pages`, `select_page`, `close_page`) | Multi-source research, crawlers, multi-step flows |
| **Interaction** (`click`, `fill_form`, `fill`, `type_text`, `press_key`, `hover`, `drag`, `upload_file`, `handle_dialog`) | QA automation, form filing, e-commerce workflows, onboarding |
| **Inspection** (`take_snapshot`, `take_screenshot`, `evaluate_script`, `list_console_messages`, `get_console_message`, `list_network_requests`, `get_network_request`) | Security audits, API documentation, error triage, content extraction |
| **Performance** (`performance_start_trace`, `performance_stop_trace`, `performance_analyze_insight`, `take_memory_snapshot`) | CI gates, memory leak hunting, third-party script analysis |
| **Audit** (`lighthouse_audit`, `emulate`, `resize_page`) | Accessibility, SEO, responsive QA, device simulation |

---

## What Makes This Different from Plain Web Scraping

| Capability | Plain scraper | chrome-devtools-mcp |
|-----------|--------------|---------------------|
| JS-rendered content | No | Yes |
| SPA navigation | No | Yes |
| Console errors | No | Yes |
| Network request bodies | No | Yes |
| Heap memory analysis | No | Yes |
| Performance traces | No | Yes |
| Device emulation | No | Yes |
| Login sessions | No | Yes |
| Native browser dialogs | No | Yes |
| Accessibility tree | No | Yes |
| Lighthouse audits | No | Yes |
| File uploads | No | Yes |
