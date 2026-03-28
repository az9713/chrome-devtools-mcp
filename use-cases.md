# chrome-devtools-mcp — AI Agentic Use Cases

A catalogue of workflows where an AI agent with access to `chrome-devtools-mcp` can do genuinely useful work. Two demos are already documented in this repo; the list below covers what else is possible across every domain.

---

## How to read this

Each use case includes:
- **What the agent does** — the concrete task
- **Key tools** — which of the 29 tools drive it
- **Why it beats alternatives** — what makes the CDP approach better than a plain web scraper or a Lighthouse run
- **Prompt** — copy-paste this into Claude Code (with chrome-devtools-mcp configured) to trigger the workflow

---

## Our two demos

### Demo 1 — Website health check (CNN.com)
Full walkthrough: [`demo-health-check/cnn-case-study/walkthrough.md`](demo-health-check/cnn-case-study/walkthrough.md)

> **Prompt:**
> "Using chrome-devtools-mcp, run a full website health check on https://www.cnn.com. Open the page in a new tab, dismiss any consent or cookie dialogs that appear, then: take a screenshot, read the accessibility tree, start a performance trace and reload the page, stop the trace and analyze Core Web Vitals (LCP, CLS, INP), list all network requests filtering for third-party domains and count how many unique vendors are loading resources, check for any console errors, and attempt a Lighthouse audit. Write a health report covering: page structure, performance scores, third-party vendor count, any forced reflow or layout thrashing issues, cache headers, and your top 5 recommendations for improvement."

---

### Demo 2 — Agentic shopping workflow (saucedemo.com)
Full walkthrough: [`demo-shopping/walkthrough.md`](demo-shopping/walkthrough.md)

> **Prompt:**
> "Using chrome-devtools-mcp, act as an autonomous shopping agent on https://www.saucedemo.com. Log in using fill_form with username standard_user and password secret_sauce. Browse the product catalog and hover over the Sauce Labs Fleece Jacket. Navigate to its detail page. Open the Sauce Labs Backpack detail page in a second background tab for comparison, then close that tab. Add all 6 products to the cart. Switch to iPhone 14 Pro mobile emulation (390×844, 3× DPR, touch enabled) and take a screenshot showing the mobile layout. Take a heap memory snapshot and save it to demo-shopping/memory-before-checkout.heapsnapshot. Navigate to the cart, proceed to checkout, fill the shipping form with first name Jane, last name Tester, zip 94107, and complete the order. Report any broken third-party integrations discovered in the network requests."

---

## 1. Quality Assurance & Testing

### 1.1 Regression snapshot testing
The agent opens every page in a sitemap, takes a full-page screenshot, and diffs it against a stored baseline. Any pixel change surfaces as a finding.

**Key tools:** `navigate_page`, `take_screenshot` (fullPage), `list_network_requests`
**Why CDP:** Screenshots include fonts, custom CSS, lazy-loaded images — things headless screenshot tools often miss.

> **Prompt:**
> "Using chrome-devtools-mcp, open each of these URLs: [URL list]. Take a full-page screenshot of each and save to ./screenshots/current/[page-name].png. Compare each against the baseline screenshots in ./screenshots/baseline/. List every page where the layout, text, or visual elements have changed, and describe what is different."

---

### 1.2 End-to-end form validation
The agent fills every form field on a page with boundary values (empty, max length, special characters, SQL fragments) and records which inputs produce error states vs. silent failures.

**Key tools:** `fill_form`, `fill`, `type_text`, `take_snapshot`, `list_console_messages`
**Why CDP:** The a11y snapshot tells the agent whether an error message appeared without needing to parse pixel positions.

> **Prompt:**
> "Using chrome-devtools-mcp, open [URL] and test every input field on the page for validation. For each field try: (1) submitting empty, (2) a 500-character string, (3) special characters including <, >, \", &, and ', (4) a SQL fragment like ' OR 1=1 --. After each attempt, read the accessibility tree for error messages and check the console for JS errors. Report which fields accept invalid input silently and which show proper validation errors."

---

### 1.3 Cross-device responsive QA
The agent loads a page at multiple viewport sizes and compares the a11y tree and screenshots at each. Detects layout reflow, hidden nav items, and overlapping elements.

**Key tools:** `emulate`, `resize_page`, `take_screenshot`, `take_snapshot`
**Why CDP:** Single browser session, no fleet of devices needed. DPR and touch emulation are accurate.

> **Prompt:**
> "Using chrome-devtools-mcp, load [URL] at each of these viewports: 375×667 (iPhone SE), 390×844 (iPhone 14 Pro), 768×1024 (iPad portrait), 1024×768 (iPad landscape), 1280×800 (laptop), 1920×1080 (desktop). At each size, emulate the appropriate device, take a screenshot, and read the accessibility tree. Report any navigation items that disappear, any content that overflows or is clipped, and any interactive elements that become unreachable at any size."

---

### 1.4 Broken link and 404 detector
The agent crawls a site, follows every `<a>` tag, and reports all links that result in a 4xx or 5xx HTTP status.

**Key tools:** `take_snapshot`, `navigate_page`, `list_network_requests`
**Why CDP:** Catches links that redirect through JS before issuing the final request — pure HTML scrapers miss these.

> **Prompt:**
> "Using chrome-devtools-mcp, open [URL] and read the accessibility tree to find all links on the page. Navigate to each link, check the HTTP status code from the network requests, then go back. List every link that returns a 4xx or 5xx status code, along with the link text and destination URL."

---

### 1.5 Dialog and modal interaction testing
The agent triggers every modal on a page (cookie banners, age gates, newsletter popups, confirm dialogs) and verifies each can be dismissed.

**Key tools:** `click`, `handle_dialog`, `take_snapshot`
**Why CDP:** `handle_dialog` intercepts native browser `alert()`/`confirm()`/`prompt()` — impossible to automate from outside the browser.

> **Prompt:**
> "Using chrome-devtools-mcp, open [URL] and find every button, link, or trigger that opens a modal, popup, overlay, or native browser dialog. Click each one, take a screenshot of the open state, read the accessibility tree, then dismiss it. Report any dialog that cannot be closed, any that blocks the rest of the page indefinitely, and any native browser dialogs that appeared."

---

### 1.6 Authentication flow testing
The agent runs through login, logout, session expiry, wrong password, and locked account flows. Records each state via snapshot and screenshot.

**Key tools:** `fill_form`, `click`, `navigate_page`, `take_snapshot`, `list_console_messages`
**Why CDP:** Can inspect network requests to verify tokens are being sent correctly alongside verifying the UI state.

> **Prompt:**
> "Using chrome-devtools-mcp, test all authentication flows on [URL]. Run these scenarios in sequence: (1) login with valid credentials [username] / [password], (2) login with wrong password, (3) login with empty fields, (4) logout after a successful login, (5) navigate directly to [protected URL] without logging in. For each scenario, take a screenshot, read the accessibility tree for error messages, and check the console for errors. Report any scenario that behaves unexpectedly."

---

## 2. Performance Engineering

### 2.1 Core Web Vitals CI gate
Before a deploy, the agent runs a performance trace, extracts LCP, CLS, and INP, and fails the build if any metric regresses beyond a threshold.

**Key tools:** `performance_start_trace`, `performance_stop_trace`, `performance_analyze_insight`
**Why CDP:** Measures real browser rendering metrics — not synthetic estimates. Catches regressions that Lighthouse misses in practice.

> **Prompt:**
> "Using chrome-devtools-mcp, run a performance audit on [staging URL]. Start a performance trace, navigate to the page, let it fully load, then stop the trace and analyze insights. Report the Core Web Vitals: LCP, CLS, and INP. If LCP exceeds 2.5 seconds, CLS exceeds 0.1, or INP exceeds 200 milliseconds, flag the check as FAILED and identify the specific element or interaction responsible."

---

### 2.2 Third-party script impact audit
The agent loads a page and quantifies the performance cost of every third-party vendor.

**Key tools:** `performance_start_trace`, `performance_analyze_insight`, `list_network_requests`, `get_network_request`
**Why CDP:** Can pinpoint which exact JS file from which domain caused the longest task on the main thread.

> **Prompt:**
> "Using chrome-devtools-mcp, load [URL] and start a performance trace. After the page finishes loading, stop the trace and list all network requests. Group every request by domain and identify which are first-party vs. third-party. For each third-party domain, report: how many requests it made, total bytes transferred, and whether any of its scripts appear in the performance trace as long tasks (>50ms). Rank third-party domains by their performance impact."

---

### 2.3 Memory leak hunter
The agent navigates through an SPA route sequence repeatedly, taking heap snapshots before and after to detect growing object counts.

**Key tools:** `take_memory_snapshot`, `navigate_page`
**Why CDP:** V8 heap snapshots are the gold standard for JS memory analysis. No other tool gives this level of detail without native DevTools.

> **Prompt:**
> "Using chrome-devtools-mcp, open [SPA URL] and take a heap snapshot — save as ./snapshots/baseline.heapsnapshot. Then navigate through this route sequence 10 times: [route 1 → route 2 → route 3 → back to start]. Take a second heap snapshot and save as ./snapshots/after-navigation.heapsnapshot. Report the file sizes of both snapshots. If the second is more than 20% larger than the first, flag a likely memory leak and note which navigation sequence to investigate."

---

### 2.4 Slow-network experience audit
The agent throttles the connection to "Slow 3G", loads a page, and records which resources block rendering and what the LCP element is.

**Key tools:** `emulate` (networkConditions), `performance_start_trace`, `performance_analyze_insight`, `take_screenshot`
**Why CDP:** Tests the real experience on slow connections without needing a physical device on a slow network.

> **Prompt:**
> "Using chrome-devtools-mcp, emulate Slow 3G network conditions, then open [URL] and start a performance trace. Let the page fully load, stop the trace, and analyze the insights. Report: time to first byte, the LCP element and how long it took to load, which resources were render-blocking, the total page weight in MB, and the total time to fully interactive. Take a screenshot showing what the page looks like at the 5-second mark."

---

### 2.5 CPU throttle regression test
The agent emulates a mid-range Android device by applying 4× CPU throttling, then measures interaction responsiveness.

**Key tools:** `emulate` (cpuThrottlingRate), `performance_start_trace`, `click`, `performance_analyze_insight`
**Why CDP:** Directly reproduces the experience of a low-end device without owning one.

> **Prompt:**
> "Using chrome-devtools-mcp, apply 4× CPU throttling to emulate a mid-range Android device, then open [URL]. Start a performance trace, click [interactive element — e.g. the main navigation menu, an accordion, a tab], and stop the trace immediately after the interaction completes. Report how long the interaction took from click to visual response, and whether any JavaScript task on the main thread exceeded 50ms during the interaction."

---

## 3. Accessibility & Compliance

### 3.1 WCAG accessibility audit pipeline
The agent runs a Lighthouse accessibility audit on every page of a site and aggregates results into a report.

**Key tools:** `lighthouse_audit`, `navigate_page`, `take_snapshot`
**Why CDP:** Lighthouse's accessibility engine runs inside Chrome — it sees the fully-rendered DOM including dynamically injected content.

> **Prompt:**
> "Using chrome-devtools-mcp, run a Lighthouse accessibility audit on each of these pages: [URLs]. For each page, report the accessibility score, list every failing audit item with its description and the affected HTML elements, and suggest a specific fix for each failure. At the end, produce a summary table sorted by score ascending (worst first)."

---

### 3.2 Keyboard-only navigation test
The agent navigates a page using only Tab, Shift+Tab, Enter, and Space, recording the focus order.

**Key tools:** `press_key`, `take_snapshot`, `take_screenshot`
**Why CDP:** The a11y tree shows `focusable` and `focused` state directly — no pixel analysis required.

> **Prompt:**
> "Using chrome-devtools-mcp, open [URL] and navigate the entire page using only keyboard input: Tab to move forward, Shift+Tab to move back, Enter to activate buttons and links, Space to toggle checkboxes. After each Tab press, read the accessibility tree to record which element has focus. Report: the complete focus order, any interactive element that is skipped by Tab, any element where focus disappears entirely, and any button or link that cannot be activated by keyboard."

---

### 3.3 Screen reader simulation
The agent traverses the accessibility tree as a screen reader would and narrates the experience.

**Key tools:** `take_snapshot` (verbose mode), `navigate_page`
**Why CDP:** The full a11y tree exposes role, label, state, and description for every node — exactly what a screen reader sees.

> **Prompt:**
> "Using chrome-devtools-mcp, open [URL] and take a verbose accessibility tree snapshot. Read through it in depth-first order as a screen reader would. Narrate what a visually impaired user would hear announced on this page. Flag any: image without alt text, button without an accessible label, form field without a name or label, heading hierarchy that skips levels (e.g. H1 → H3), and decorative element that would be read aloud unnecessarily."

---

### 3.4 Cookie consent compliance checker
The agent loads a page with no cookies and records which third-party requests fire before the user consents.

**Key tools:** `navigate_page`, `list_network_requests`, `take_snapshot`, `click`
**Why CDP:** Captures the precise moment each network request fires relative to user interaction — essential for GDPR pre-consent evidence.

> **Prompt:**
> "Using chrome-devtools-mcp, open [URL] in a fresh session with no prior cookies. Immediately list all network requests that have fired before any user interaction. Read the accessibility tree to identify the cookie consent banner. List every third-party domain that sent a request before the consent banner was accepted — these may be GDPR violations. Then click the consent button and note which additional requests fire afterwards."

---

## 4. SEO & Content

### 4.1 SEO technical audit
The agent runs a Lighthouse SEO audit, then augments it with additional checks for canonical tags, title length, meta descriptions, and hreflang.

**Key tools:** `lighthouse_audit`, `navigate_page`, `evaluate_script`
**Why CDP:** Sees the page after JS has executed — catches tags injected by React/Next/Nuxt that a static crawler would miss.

> **Prompt:**
> "Using chrome-devtools-mcp, run a Lighthouse SEO audit on [URL], then run these additional checks using evaluate_script: (1) is there exactly one canonical tag?, (2) is the title tag under 60 characters?, (3) is the meta description under 160 characters?, (4) are there any hreflang tags?, (5) is there an H1 tag and is it unique?. Produce a complete SEO report with pass/fail for each check and specific fix instructions for any failures."

---

### 4.2 Structured data validator
The agent extracts all JSON-LD and microdata from a page and validates required fields against schema.org.

**Key tools:** `evaluate_script`, `navigate_page`
**Why CDP:** Runs inside the browser — catches structured data injected after page load by tag managers.

> **Prompt:**
> "Using chrome-devtools-mcp, open [URL] and use evaluate_script to extract all structured data: every JSON-LD script block and any microdata attributes (itemtype, itemprop). For each schema type found (Product, Article, BreadcrumbList, Organization, etc.), list all present fields and identify any required fields that are missing according to schema.org. Also flag any field with an incorrect data type (e.g. price as a string instead of a number)."

---

### 4.3 Competitor content tracker
The agent visits competitor pages, extracts key content, and diffs it against a previous capture.

**Key tools:** `navigate_page`, `take_snapshot`, `evaluate_script`
**Why CDP:** Renders JS-heavy pages correctly — doesn't miss content that a plain HTTP scraper would return as empty.

> **Prompt:**
> "Using chrome-devtools-mcp, visit each of these competitor pages: [URLs]. For each page, extract: the page title, H1 heading, the first 3 product or feature descriptions visible on the page, and any pricing information. Take a screenshot of each. Compare with this previous data: [paste previous snapshot]. List everything that has changed since the last check."

---

### 4.4 Meta tag completeness checker
The agent checks every page in a sitemap for Open Graph, Twitter Card, canonical, and meta description tags.

**Key tools:** `navigate_page`, `evaluate_script`
**Why CDP:** Works on SPAs where meta tags are injected by client-side frameworks.

> **Prompt:**
> "Using chrome-devtools-mcp, visit each of these URLs: [URLs]. On each page, use evaluate_script to check for the presence of: title tag, meta description, og:title, og:description, og:image, og:url, twitter:card, twitter:title, twitter:image, and canonical link tag. Produce a table with each URL as a row and each tag as a column, marking present as ✓ and missing as ✗. List any page missing more than two tags at the bottom as a priority fix."

---

## 5. E-commerce & Retail

### 5.1 Price monitoring agent
The agent visits product pages, extracts current prices and stock status, and alerts on changes.

**Key tools:** `navigate_page`, `evaluate_script`, `take_screenshot`
**Why CDP:** Handles sites that require JS rendering to show the real price (after personalisation, geolocation, or login).

> **Prompt:**
> "Using chrome-devtools-mcp, open each of these product pages: [URLs]. For each page, extract the current price, any sale price or badge, and the stock availability status. Take a screenshot. Compare with these baseline prices: [product: price list]. Report any product where the price has dropped by more than 10%, any that has a sale badge that wasn't there before, and any out-of-stock item that is now available."

---

### 5.2 Checkout funnel QA
The agent runs a complete purchase flow on staging before every deploy, verifying UI state and API payloads at each step.

**Key tools:** `fill_form`, `click`, `evaluate_script`, `list_network_requests`, `get_network_request`
**Why CDP:** Inspects the actual API calls made during checkout to verify the right payload is being sent, not just that the UI looks correct.

> **Prompt:**
> "Using chrome-devtools-mcp, run through the complete checkout flow on [staging URL]. Add [product name] to the cart, apply discount code [code], fill in shipping details with name [name], address [address], city [city], zip [zip], select standard shipping, and complete the order. At each step, take a screenshot, verify the UI shows the expected state, and inspect the network requests to confirm the correct API endpoint was called with the right payload. Report any step where the UI and the API response diverge."

---

### 5.3 Discount code validator
The agent tests a batch of promo codes on a checkout page and reports which are valid, expired, or invalid.

**Key tools:** `fill`, `click`, `take_snapshot`, `evaluate_script`

> **Prompt:**
> "Using chrome-devtools-mcp, open the checkout page on [URL] and add [product] to the cart first if needed. Then test each of these discount codes one at a time: [code list]. For each code: enter it in the promo/discount field, read the accessibility tree to get the result message (discount applied, expired, invalid, etc.), take a screenshot, then clear the field before trying the next code. Produce a table showing each code, its status, and the discount amount or error message."

---

### 5.4 Product catalogue crawler
The agent pages through an entire product catalogue and exports structured data for every item.

**Key tools:** `navigate_page`, `take_snapshot`, `evaluate_script`, `click`
**Why CDP:** Handles infinite scroll, "load more" buttons, and session-gated catalogues.

> **Prompt:**
> "Using chrome-devtools-mcp, crawl the product catalogue starting at [URL]. On each listing page, use evaluate_script to extract the name, price, SKU, and image URL for every product card. Then find and click the 'Next page' or 'Load more' button and repeat, until no pagination button remains. Return the complete data as a JSON array, and report the total number of products found."

---

### 5.5 Flash sale monitor
The agent checks a product page for a price drop or sale badge and screenshots the current offer state.

**Key tools:** `navigate_page`, `evaluate_script`, `take_screenshot`

> **Prompt:**
> "Using chrome-devtools-mcp, open [product URL] and check: (1) is the current price lower than [target price]?, (2) is there any element on the page containing the text 'Sale', 'Flash deal', 'Limited time', or 'X% off'?. Take a screenshot of the current price and any badge. Report the current price, whether the target price has been reached, and any promotional text visible on the page."

---

## 6. Research & Information Gathering

### 6.1 Multi-source research synthesiser
The agent opens multiple sources in parallel tabs, reads each, and synthesises a structured briefing.

**Key tools:** `new_page`, `list_pages`, `select_page`, `take_snapshot`, `close_page`
**Why CDP:** Multi-tab management lets the agent hold a reading list in parallel and switch between sources without losing state.

> **Prompt:**
> "Using chrome-devtools-mcp, open each of these URLs in separate tabs: [URLs]. For each tab, select it, read the accessibility tree to extract the main content, take a screenshot, then close the tab. After reading all sources, synthesise the information into a structured briefing covering: the key claims or findings from each source, areas where sources agree, and areas where they contradict each other."

---

### 6.2 Academic literature review
The agent searches a research database, opens each result, and produces an annotated bibliography.

**Key tools:** `navigate_page`, `fill`, `press_key`, `take_snapshot`, `evaluate_script`

> **Prompt:**
> "Using chrome-devtools-mcp, go to https://pubmed.ncbi.nlm.nih.gov, search for '[search term]', and open the first 10 results each in a new tab. From each paper's page extract: title, authors, publication year, journal name, and abstract. Close each tab when done. Produce an annotated bibliography sorted by publication year descending, with a one-sentence summary of each paper's main finding and its relevance to [research topic]."

---

### 6.3 Job market research
The agent searches multiple job boards and produces a salary benchmark and skills frequency report.

**Key tools:** `navigate_page`, `fill_form`, `take_snapshot`, `evaluate_script`

> **Prompt:**
> "Using chrome-devtools-mcp, search for '[job title]' jobs in '[city/remote]' on LinkedIn Jobs, Indeed, and Glassdoor. From each site, extract the job title, company name, salary range (if shown), and required skills listed in the first 10 results. Produce a report showing: the average and range of salaries across all results, the 10 most frequently required skills, and which companies are posting the most openings."

---

### 6.4 Real estate listing aggregator
The agent collects matching property listings from multiple sites and builds a comparison table.

**Key tools:** `navigate_page`, `fill_form`, `take_snapshot`, `evaluate_script`, `list_pages`, `select_page`

> **Prompt:**
> "Using chrome-devtools-mcp, search for properties on Zillow and Redfin with these criteria: location [city/neighbourhood], minimum [X] bedrooms, price range [min]–[max]. From each listing, extract address, list price, square footage, bedrooms, bathrooms, and days on market. Open listings in separate tabs as needed and close when done. Return a comparison table sorted by price-per-square-foot ascending, and flag any property that has a different price on Zillow versus Redfin."

---

### 6.5 Flight and hotel price matrix
The agent checks booking sites for multiple routes and dates and returns the cheapest options.

**Key tools:** `navigate_page`, `fill_form`, `click`, `take_snapshot`, `evaluate_script`

> **Prompt:**
> "Using chrome-devtools-mcp, check flight prices on Google Flights for these routes and dates: [route 1: origin → destination, dates], [route 2], [route 3]. For each route, extract the cheapest available fare, airline name, number of stops, and total travel time. Also check hotel prices on Booking.com near the arrival airport for the same dates, extracting the cheapest 3-star or higher option. Return a complete trip cost matrix for all routes."

---

## 7. Developer Tooling & DevOps

### 7.1 Storybook component screenshot library
The agent visits every story in a Storybook instance and saves a screenshot of each.

**Key tools:** `navigate_page`, `take_snapshot`, `take_screenshot`
**Why CDP:** Storybook renders in a real browser — components look exactly as they do in production.

> **Prompt:**
> "Using chrome-devtools-mcp, open the Storybook instance at [URL]. Read the accessibility tree to find the story navigation sidebar. Click every story link systematically — working through each component group. For each story, take a screenshot and save it to ./storybook-screenshots/[ComponentName]/[StoryName].png. At the end, list all screenshot file paths produced."

---

### 7.2 API response inspector
The agent navigates an authenticated web app, triggers actions, and documents the underlying API calls.

**Key tools:** `navigate_page`, `click`, `list_network_requests`, `get_network_request`
**Why CDP:** Captures the full request body, response body, and all headers — including auth tokens — without needing API docs.

> **Prompt:**
> "Using chrome-devtools-mcp, open [web app URL] and log in with [credentials]. Perform each of these actions: [action 1], [action 2], [action 3]. After each action, list the XHR and fetch network requests that fired. For each request, show the full URL, HTTP method, request headers, request body, response status code, and response body. I want to document the internal API endpoints this app is calling."

---

### 7.3 Console error triage
The agent loads a list of URLs, collects all console errors, de-duplicates, and produces a prioritised fix list.

**Key tools:** `navigate_page`, `list_console_messages`, `get_console_message`
**Why CDP:** Captures runtime JS errors that only appear after user interaction — not just on initial load.

> **Prompt:**
> "Using chrome-devtools-mcp, visit each of these URLs: [URLs]. On each page, read all console messages and filter for errors and warnings. Also click the main interactive elements on each page (primary CTA, navigation menu, any forms) and check for errors that appear after interaction. Collect all errors across all pages, de-duplicate by error message, and produce a prioritised list showing: error message, how many pages it affects, the list of affected URLs, and a suggested fix."

---

### 7.4 Feature flag verifier
The agent loads a page under different conditions and documents which UI elements appear in each state.

**Key tools:** `navigate_page`, `emulate` (geolocation), `take_snapshot`, `evaluate_script`

> **Prompt:**
> "Using chrome-devtools-mcp, open [URL] in three states: (1) without logging in, (2) logged in as [user A with credentials], (3) logged in as [user B with credentials]. For each state, read the accessibility tree and take a screenshot. Compare the three accessibility trees and list every UI element that appears in one state but not another — these are likely feature flags or permission-gated features."

---

### 7.5 Localisation QA
The agent loads a page with different locale settings and checks for untranslated strings and layout overflow.

**Key tools:** `emulate` (userAgent), `navigate_page`, `take_snapshot`, `take_screenshot`

> **Prompt:**
> "Using chrome-devtools-mcp, open [URL] four times, each time emulating a different locale by setting the user agent and Accept-Language: (1) English (en-US), (2) French (fr-FR), (3) German (de-DE), (4) Japanese (ja-JP). For each locale, take a screenshot and read the accessibility tree. Report: whether the page language changed for each locale, any English text still visible on non-English pages (untranslated strings), and any text that overflows its container due to longer translated content."

---

### 7.6 Dark mode visual audit
The agent screenshots every page in light and dark mode, flagging contrast failures.

**Key tools:** `emulate` (colorScheme), `take_screenshot`, `navigate_page`

> **Prompt:**
> "Using chrome-devtools-mcp, open [URL], emulate dark mode using the colorScheme setting, take a full-page screenshot and save as ./dark-mode.png. Then switch to light mode and take another full-page screenshot saved as ./light-mode.png. Compare the two and report any element that appears to have very low contrast in dark mode — specifically look for white or light text on white backgrounds, or dark text that blends into the dark theme. List affected elements with their visible text."

---

## 8. Content Creation & Documentation

### 8.1 Automated tutorial screenshot generator
The agent runs through a user flow step by step, saving a numbered screenshot at every action.

**Key tools:** `navigate_page`, `fill_form`, `click`, `take_screenshot`
**Why CDP:** Screenshots are taken at the exact right moment — after state changes settle — not mid-transition.

> **Prompt:**
> "Using chrome-devtools-mcp, walk through this user flow on [URL] and save a numbered screenshot after each step: [step 1: describe action], [step 2: describe action], [step 3: describe action]... Save each screenshot as ./tutorial/step-01.png, step-02.png, etc. After each action, wait for the UI to settle before screenshotting. At the end, list all files produced with a one-sentence caption for each describing what it shows."

---

### 8.2 Release notes screenshot pack
The agent visits every changed page on staging and captures desktop and mobile screenshots.

**Key tools:** `navigate_page`, `take_screenshot`, `emulate`

> **Prompt:**
> "Using chrome-devtools-mcp, visit each of these pages on our staging environment: [URLs]. For each page, take a full-page desktop screenshot (1280×800) and save as ./release-notes/[feature-name]-desktop.png. Then emulate an iPhone 14 Pro (390×844, mobile, touch) and take a second screenshot saved as ./release-notes/[feature-name]-mobile.png. List all files produced."

---

### 8.3 Website-to-PDF report generator
The agent navigates a multi-page dashboard and captures full-page screenshots of every section.

**Key tools:** `navigate_page`, `take_screenshot` (fullPage), `evaluate_script`

> **Prompt:**
> "Using chrome-devtools-mcp, log into [dashboard URL] with [credentials]. Navigate to each of these report sections: [section list]. On each section, take a full-page screenshot and save to ./report-[today's date]/[section-name].png. At the end, list all files produced in the order they should appear in the report, with the section name and URL for each."

---

### 8.4 Design-to-implementation comparison
The agent screenshots the live implementation at multiple viewports for comparison against design files.

**Key tools:** `take_screenshot`, `navigate_page`, `emulate`

> **Prompt:**
> "Using chrome-devtools-mcp, open [URL] and take a full-page desktop screenshot at 1440×900 — save as ./design-check/desktop.png. Then emulate an iPhone 14 Pro (390×844) and take a mobile screenshot — save as ./design-check/mobile.png. Then emulate an iPad (768×1024) and save as ./design-check/tablet.png. I will compare these against the Figma designs and note any spacing, colour, typography, or layout differences."

---

## 9. Security & Compliance

### 9.1 Exposed secrets scanner
The agent inspects inline JS, network payloads, and console output for API keys, tokens, and credentials.

**Key tools:** `navigate_page`, `evaluate_script`, `list_network_requests`, `get_network_request`, `list_console_messages`
**Why CDP:** Sees the fully evaluated JS, including strings that are only assembled at runtime.

> **Prompt:**
> "Using chrome-devtools-mcp, load each of these URLs: [URLs]. On each page, use evaluate_script to search all inline script tag contents and window-level JS variables for strings matching these patterns: sk_live_, pk_live_, AKIA, ghp_, eyJ (JWT prefix), any string over 30 characters that is hex or base64. Also inspect all network request bodies for the same patterns. List every match found with its location (inline script, network request URL, variable name) and the partial value (first/last 4 chars only)."

---

### 9.2 Mixed content detector
The agent loads HTTPS pages and flags any resource loaded over insecure HTTP.

**Key tools:** `navigate_page`, `list_network_requests`
**Why CDP:** Captures every sub-resource request including those triggered by third-party scripts.

> **Prompt:**
> "Using chrome-devtools-mcp, open each of these HTTPS pages: [URLs]. After each page loads, list all network requests and filter for any whose URL starts with 'http://' rather than 'https://'. For each insecure request found, report: the resource type (script, image, stylesheet, iframe, fetch), the full URL, and the likely origin (first-party code or a third-party tag). These are mixed content violations."

---

### 9.3 CSP violation reporter
The agent loads pages and reads Content Security Policy violation messages from the browser console.

**Key tools:** `navigate_page`, `list_console_messages`, `get_console_message`
**Why CDP:** CSP violations are emitted as native browser console events — only visible from inside the browser.

> **Prompt:**
> "Using chrome-devtools-mcp, open [URL] and read all console messages. Filter for any message that contains 'Content Security Policy', 'CSP', 'Refused to load', or 'Refused to execute'. For each violation found, report: the exact console message, the blocked URL or inline script, and the CSP directive that was violated. Then suggest the specific CSP rule change needed to allow each legitimate resource while keeping malicious ones blocked."

---

### 9.4 Analytics event verifier
The agent performs user actions and verifies the correct analytics events fired with the correct properties.

**Key tools:** `click`, `fill_form`, `list_network_requests`, `get_network_request`
**Why CDP:** Can filter for XHR/fetch requests to known analytics endpoints and inspect the exact payload.

> **Prompt:**
> "Using chrome-devtools-mcp, open [URL] and perform these user actions one at a time: [action 1 — e.g. click the 'Sign Up' CTA], [action 2 — e.g. submit the contact form], [action 3 — e.g. click a product]. After each action, list network requests to analytics endpoints (google-analytics.com, gtm, segment.io, mixpanel.com, amplitude). For each analytics call, show the full request payload and confirm it contains: event name [expected name], and these properties [property: expected value]. Flag any action where the expected event did not fire or the payload was missing required properties."

---

## 10. Business Operations & Automation

### 10.1 SaaS onboarding automation
The agent signs up for a free trial, completes onboarding, and produces a product summary.

**Key tools:** `fill_form`, `click`, `handle_dialog`, `take_snapshot`, `take_screenshot`

> **Prompt:**
> "Using chrome-devtools-mcp, sign up for a free trial on [SaaS URL]. Use these details: name [name], email [email@example.com], company [company], role [role]. Complete every step of the onboarding wizard — take a screenshot at each screen and accept any confirmation dialogs that appear. After onboarding is complete, summarise: what the product does in plain English, what integrations or connections it asks you to set up, what the first action it encourages new users to take is, and what the upgrade prompt says."

---

### 10.2 Invoice and form filing automation
The agent logs into a portal, fills a recurring form, uploads a file, and submits.

**Key tools:** `fill_form`, `fill`, `click`, `press_key`, `upload_file`, `handle_dialog`

> **Prompt:**
> "Using chrome-devtools-mcp, log into [portal URL] with username [user] and password [password]. Navigate to [form section name]. Fill in the form with these values: [field name: value, field name: value...]. Upload the file at [local file path] to the attachment field. Submit the form. If any confirmation dialog appears, accept it. Take a screenshot of the confirmation or success page and report the reference number or confirmation text shown."

---

### 10.3 Competitor feature matrix builder
The agent visits pricing pages for multiple competitors and extracts the feature list for each tier.

**Key tools:** `navigate_page`, `take_snapshot`, `evaluate_script`, `new_page`, `list_pages`, `select_page`

> **Prompt:**
> "Using chrome-devtools-mcp, open the pricing or features page for each of these companies in separate tabs: [company: URL list]. For each page, extract: the product tier names (e.g. Free, Pro, Enterprise), the features listed under each tier, and the price if shown. Close each tab when done. Produce a comparison matrix with companies as columns and all unique features as rows, marking each cell as included, excluded, or enterprise-only."

---

### 10.4 Job application assistant
The agent fills an application form using a stored candidate profile, uploads a CV, and screenshots the confirmation.

**Key tools:** `fill_form`, `upload_file`, `click`, `take_screenshot`

> **Prompt:**
> "Using chrome-devtools-mcp, open the job application form at [URL]. Fill in the following fields: first name [name], last name [surname], email [email], phone [number], LinkedIn profile URL [URL], years of experience [X], cover letter [paste text]. Upload the CV file at [local file path] to the resume upload field. Submit the application. Take a screenshot of the confirmation page and report the confirmation message and any application reference number shown."

---

### 10.5 Social proof aggregator
The agent collects recent reviews from multiple platforms and produces a sentiment summary.

**Key tools:** `navigate_page`, `take_snapshot`, `evaluate_script`

> **Prompt:**
> "Using chrome-devtools-mcp, visit the review pages for [product/company name] on G2 ([URL]), Capterra ([URL]), and Trustpilot ([URL]). From each site extract: the overall rating, total review count, and the 5 most recent reviews (reviewer name, rating out of 5, review text, date posted). Produce a combined report showing: average rating across all platforms, the most frequently mentioned positive themes, the most frequently mentioned complaints, and 3 standout quotes (one strong positive, one constructive, one mixed)."

---

### 10.6 Regulatory filing monitor
The agent checks government filing portals for new submissions from a watchlist of companies.

**Key tools:** `navigate_page`, `fill_form`, `take_snapshot`, `evaluate_script`

> **Prompt:**
> "Using chrome-devtools-mcp, go to https://efts.sec.gov/LATEST/search-index?q=%22[company name]%22&dateRange=custom&startdt=[start date]&enddt=[today] and extract the 5 most recent SEC filings: form type, date filed, and description. Take a screenshot of the results. Compare with these previously known filings: [list]. Highlight any filing that is new since the last check and summarise what type of disclosure each new filing represents."

---

## 11. Personal Productivity

### 11.1 Daily briefing builder
The agent reads a custom list of sources each morning and assembles a digest.

**Key tools:** `new_page`, `navigate_page`, `take_snapshot`, `evaluate_script`, `close_page`

> **Prompt:**
> "Using chrome-devtools-mcp, open each of these pages in separate tabs: [news site URLs, newsletter URLs, dashboard URLs]. For each tab, read the accessibility tree to extract the main headlines, key numbers, or important updates. Close each tab when done. Assemble a morning briefing with sections for: top news headlines, any market or metric summaries, and any alerts or action items from the dashboards."

---

### 11.2 Grocery order automator
The agent logs into a grocery delivery site, adds a weekly list to the basket, and completes the order.

**Key tools:** `fill`, `click`, `evaluate_script`, `handle_dialog`, `take_screenshot`

> **Prompt:**
> "Using chrome-devtools-mcp, log into [grocery site URL] with username [user] and password [password]. Search for each of these items and add the first result to the basket: [item list]. Once all items are in the basket, navigate to checkout. Apply discount code [code] if available. Confirm the order total, complete the purchase, and take a screenshot of the order confirmation page. If any confirmation dialog appears asking to confirm the order, accept it."

---

### 11.3 Portfolio tracker
The agent logs into brokerage dashboards and aggregates holdings into a net-worth summary.

**Key tools:** `navigate_page`, `fill_form`, `evaluate_script`, `take_snapshot`
**Why CDP:** Works on sites behind login and with client-side-rendered data that no public API exposes.

> **Prompt:**
> "Using chrome-devtools-mcp, log into [brokerage 1 URL] with username [user] and password [password]. Navigate to the portfolio overview page and extract: each holding's name, ticker symbol, quantity, current price, total value, and today's gain/loss. Take a screenshot. Then do the same for [brokerage 2 URL]. Add up the total portfolio value across both accounts, calculate today's net gain/loss, and produce a consolidated one-page summary."

---

### 11.4 Meeting prep researcher
The agent researches a company and attendees before a meeting and produces a briefing.

**Key tools:** `navigate_page`, `take_snapshot`, `evaluate_script`

> **Prompt:**
> "Using chrome-devtools-mcp, I have a meeting with people from [company]. Open [company website URL] and extract: what they do, their key products or services, company size if listed, and any recent news or announcements. Then open [LinkedIn URL for person 1] and extract their current title, previous roles, and any notable background. Do the same for [LinkedIn URL for person 2]. Produce a one-page meeting brief with a company overview paragraph and a one-paragraph bio for each person."

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
