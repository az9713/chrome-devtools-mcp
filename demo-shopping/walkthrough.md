# Saucedemo.com — Agentic Shopping Workflow Walkthrough

Full annotated step-by-step of every tool used, every input/output, and every obstacle encountered.

---

## Setup

**Target:** https://www.saucedemo.com — a purpose-built fake e-commerce site by Sauce Labs, designed for browser automation demos. No real payments; no real data.

**Goal:** Run a complete purchase flow from login to order confirmation using as many unused chrome-devtools-mcp tools as possible. The CNN health-check demo used 11/29 tools; this demo targets the remaining 18.

**Credentials:**
- Username: `standard_user`
- Password: `secret_sauce`

---

## Phase 1: Open Browser + Login

### Tool 1 — `new_page`

**What it is:** Opens a new browser tab and navigates to a URL.

**What it does:** Creates an isolated tab in the Chrome instance connected via CDP. Returns the list of all open pages with their IDs.

**How it works internally:** Sends `Target.createTarget` to Chrome CDP, then `Page.navigate` to load the URL. The new tab becomes the "selected" page for subsequent tool calls.

**Input:**
```json
{ "url": "https://www.saucedemo.com" }
```

**Output:**
```
Pages
1: about:blank
2: https://www.saucedemo.com/ [selected]
```

**What it told us:** Tab 2 opened and selected. The login page loaded immediately — no redirects, no SSL issues.

---

### Tool 2 — `take_snapshot`

**What it is:** Returns a text representation of the page's accessibility tree (a11y tree).

**What it does:** Traverses every element Chrome's accessibility engine can see and returns them as a tree with unique `uid` identifiers. These UIDs are required by all interaction tools (`click`, `fill`, `hover`, etc.).

**How it works internally:** Calls `Accessibility.getFullAXTree` via CDP, filters out invisible/ignored nodes, and serialises the result as indented text.

**Input:** _(no parameters)_

**Output (abbreviated):**
```
uid=1_0 RootWebArea "Swag Labs"
  uid=1_2 textbox "Username"
  uid=1_3 textbox "Password"
  uid=1_4 button "Login"
```

**What it told us:** Login form has 3 nodes: Username textbox (uid=1_2), Password textbox (uid=1_3), Login button (uid=1_4). Clean, minimal a11y tree — no consent dialogs, no overlays.

---

### Tool 3 — `fill_form`

**What it is:** Fills multiple form elements in a single call.

**What it does:** Takes an array of `{uid, value}` pairs and fills each field. More efficient than calling `fill` twice.

**How it works internally:** For each element, resolves the DOM node from the uid, sets its value, and dispatches a change event. For standard HTML inputs, this works reliably. For React controlled inputs, it may not trigger the synthetic event system (see obstacle below).

**Input:**
```json
{
  "elements": [
    { "uid": "1_2", "value": "standard_user" },
    { "uid": "1_3", "value": "secret_sauce" }
  ]
}
```

**Output:** `Successfully filled out the form`

**What it told us:** Both fields accepted values. The form was ready to submit.

---

### Tool 4 — `click` (Login button)

**What it is:** Clicks a page element by its uid.

**What it does:** Dispatches a synthetic mouse click at the element's centre coordinates.

**How it works internally:** CDP `Input.dispatchMouseEvent` with `type: mousePressed` then `mouseReleased` at the element's bounding box centre.

**Input:**
```json
{ "uid": "1_4" }
```

**Output:** `Successfully clicked on the element`

**Result:** Navigated to `/inventory.html`. All 6 products visible. Cart badge empty.

> **Note:** This `click` worked because the Login button's handler is a standard form submit, not a React `onClick` — no synthetic event quirk here.

---

## Phase 2: Browse Products

### Tool 5 — `list_network_requests` (filtered)

**What it is:** Returns all network requests made since the last page navigation.

**What it does:** Lists requests with method, URL, and HTTP status. Can be filtered by resource type.

**How it works internally:** Reads Chrome's internal network log via `Network.getResponseBody` and the CDP network event stream captured since navigation.

**Input:**
```json
{ "resourceTypes": ["xhr", "fetch", "document"], "pageSize": 10 }
```

**Output:**
```
reqid=1  GET  https://www.saucedemo.com/             [200]
reqid=8  POST https://events.backtrace.io/api/unique-events/submit?universe=UNIVERSE&token=TOKEN  [401]
reqid=9  POST https://events.backtrace.io/api/summed-events/submit?universe=UNIVERSE&token=TOKEN  [401]
reqid=14 POST https://events.backtrace.io/api/unique-events/submit?universe=UNIVERSE&token=TOKEN  [401]
reqid=15 POST https://events.backtrace.io/api/summed-events/submit?universe=UNIVERSE&token=TOKEN  [401]
```

**What it told us:** Every analytics ping to backtrace.io is failing 401. The token is a literal placeholder `?universe=UNIVERSE&token=TOKEN`. The app is entirely blind to errors in production.

---

### Tool 6 — `get_network_request`

**What it is:** Fetches full details of a specific request: headers, request body, response headers, response body.

**What it does:** Drills into reqid=8 to see exactly what the app is sending and what the server returned.

**How it works internally:** Calls `Network.getResponseBody` and reconstructs headers from the CDP network event stream for the given request ID.

**Input:**
```json
{ "reqid": 8 }
```

**Output (key parts):**

Request body (sent by the app):
```json
{
  "application": "Swag Store",
  "appversion": "3.0.0",
  "unique_events": [{
    "timestamp": 1774662484,
    "attributes": {
      "backtrace.agent": "@backtrace-labs/react",
      "backtrace.version": "0.0.5",
      "browser.name": "Chrome",
      "browser.version": "146.0.0.0"
    }
  }]
}
```

Response body (from backtrace.io):
```json
{ "error": { "message": "Unauthorized request", "code": 6 } }
```

**What it told us:** The app is running `@backtrace-labs/react v0.0.5`, calls itself `Swag Store v3.0.0`, and tries to send session telemetry on every page load. The `universe=UNIVERSE&token=TOKEN` in the URL are never replaced with real values — this is a misconfigured SDK integration.

---

### Tool 7 — `list_console_messages`

**What it is:** Returns all console output (log, warn, error, debug) since the last navigation.

**What it does:** Reads Chrome's console message buffer.

**How it works internally:** CDP `Runtime.consoleAPICalled` events captured since navigation.

**Input:**
```json
{ "types": ["error", "warn", "log"] }
```

**Output:** `<no console messages found>`

**What it told us:** React SPA is running cleanly. No JavaScript errors, deprecation warnings, or debug logs. The 401s happen silently at the network layer without surfacing to the console.

---

### Tool 8 — `hover`

**What it is:** Moves the mouse cursor over an element.

**What it does:** Triggers `mouseenter`, `mouseover`, and `mousemove` events. Useful for revealing hover states, tooltips, and CSS transitions.

**How it works internally:** CDP `Input.dispatchMouseEvent` with `type: mouseMoved` to the element's centre coordinates.

**Input:**
```json
{ "uid": "2_35" }
```
_(uid=2_35 = Sauce Labs Fleece Jacket product image)_

**Output:** `Successfully hovered over the element`

**What it told us:** Mouse position was moved to the Fleece Jacket image. No hover tooltip appeared (saucedemo has no hover effects), but the browser did scroll the page to bring the jacket into the lower viewport — confirming the hover coordinates were applied.

---

## Phase 3: Multi-Tab Product Comparison

### Tool 9 — `navigate_page`

**What it is:** Navigates the current page to a URL, or goes back/forward in history, or reloads.

**What it does:** Loads a new URL in the selected tab.

**How it works internally:** CDP `Page.navigate` with the given URL. Waits for `Page.loadEventFired`.

**Input:**
```json
{ "type": "url", "url": "https://www.saucedemo.com/inventory-item.html?id=5" }
```

**Output:** `Successfully navigated to https://www.saucedemo.com/inventory-item.html?id=5`

**What it told us:** Fleece Jacket detail page loaded directly. The SPA's routing accepted deep-link navigation — no redirect back to login, confirming the session cookie is still active.

> **Note on product IDs:** saucedemo uses numeric IDs in the URL (`?id=5` = Fleece Jacket, `?id=4` = Backpack). These aren't shown in the UI but are discoverable by inspecting the network requests or page source.

---

### Tool 10 — `new_page` (background tab)

**What it is:** Opens a second tab without switching to it.

**What it does:** Creates a new tab in the background; the current tab stays selected.

**Input:**
```json
{ "url": "https://www.saucedemo.com/inventory-item.html?id=4", "background": true }
```

**Output:**
```
Pages
1: about:blank
2: https://www.saucedemo.com/inventory-item.html?id=5
3: https://www.saucedemo.com/inventory-item.html?id=4 [selected]
```

> **Note:** Even with `background: true`, the new tab became `[selected]` in the output — an MCP server quirk. We immediately corrected with `select_page`.

---

### Tool 11 — `list_pages`

**What it is:** Lists all open tabs in the browser.

**What it does:** Returns tab IDs and URLs for every open page.

**How it works internally:** CDP `Target.getTargets` filtered to `page` targets.

**Input:** _(no parameters)_

**Output:**
```
1: about:blank
2: https://www.saucedemo.com/inventory-item.html?id=5
3: https://www.saucedemo.com/inventory-item.html?id=4 [selected]
```

**What it told us:** 3 tabs open. Tab IDs are stable integers assigned at creation time.

---

### Tool 12 — `select_page`

**What it is:** Sets the active page context for all subsequent tool calls.

**What it does:** Switches the "currently selected page" pointer. The selected page is where `click`, `fill`, `take_snapshot`, etc. operate.

**How it works internally:** Updates the server's internal page reference; calls `Target.activateTarget` to bring the tab to the front if `bringToFront: true`.

**Input:**
```json
{ "pageId": 2, "bringToFront": true }
```

**Output:**
```
2: https://www.saucedemo.com/inventory-item.html?id=5 [selected]
```

**What it told us:** We are now operating on the Fleece Jacket tab.

---

### Tool 13 — `evaluate_script` (add to cart — React workaround)

**What it is:** Executes arbitrary JavaScript in the page context.

**What it does:** Runs a function and returns the JSON-serialised result. The escape hatch for anything the other tools can't do.

**How it works internally:** CDP `Runtime.callFunctionOn` on the page's main frame context.

**Input:**
```javascript
() => {
  const btn = document.querySelector('button[data-test="add-to-cart"]');
  if (btn) { btn.click(); return 'clicked: ' + btn.getAttribute('data-test'); }
  return 'button not found';
}
```

**Output:** `"clicked: add-to-cart"`

**What it told us:** The `click()` call via `evaluate_script` *does* fire React's synthetic event system (because React attaches its event delegation to the document root, and `element.click()` from inside the page context properly bubbles). The "Add to cart" button changed to "Remove" and the cart badge incremented to 1.

> **Key insight:** The difference between `mcp__chrome-devtools__click` (CDP `Input.dispatchMouseEvent`) and `evaluate_script` calling `element.click()`:
> - CDP mouse events simulate physical input at coordinates; React's synthetic event system sometimes doesn't respond to these.
> - `element.click()` from within the page's JS context fires a trusted DOM event that React's delegation *does* pick up.

---

### Tool 14 — `close_page`

**What it is:** Closes a tab by its page ID.

**What it does:** Destroys the tab and removes it from the page list.

**How it works internally:** CDP `Target.closeTarget`.

**Input:**
```json
{ "pageId": 3 }
```

**Output:**
```
Pages
1: about:blank
2: https://www.saucedemo.com/inventory-item.html?id=5 [selected]
```

**What it told us:** The Backpack comparison tab is gone. We're back to 2 tabs.

---

## Phase 4: Fill the Cart

After navigating back to `/inventory.html`, we added all remaining 5 products using a single `evaluate_script` call:

```javascript
() => {
  const buttons = [...document.querySelectorAll('button[data-test^="add-to-cart"]')];
  buttons.forEach(b => b.click());
  return `clicked ${buttons.length} buttons: ${buttons.map(b => b.getAttribute('data-test')).join(', ')}`;
}
```

**Output:** `"clicked 5 buttons: add-to-cart-sauce-labs-backpack, add-to-cart-sauce-labs-bike-light, add-to-cart-sauce-labs-bolt-t-shirt, add-to-cart-sauce-labs-onesie, add-to-cart-test.allthethings()-t-shirt-(red)"`

Cart badge jumped to **6**.

---

## Phase 5: Mobile Emulation

### Tool 15 — `emulate`

**What it is:** Changes the browser's device emulation settings.

**What it does:** Overrides viewport size, device pixel ratio, touch capabilities, and user-agent string to simulate a specific device.

**How it works internally:** CDP `Emulation.setDeviceMetricsOverride` + `Emulation.setUserAgentOverride` + `Emulation.setTouchEmulationEnabled`.

**Input:**
```json
{
  "viewport": "390x844x3,mobile,touch",
  "userAgent": "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1"
}
```

**Output:** `Emulating viewport: {"deviceScaleFactor":3,"isMobile":true,"hasTouch":true,"isLandscape":false,"width":390,"height":844}`

**What it told us:** Chrome is now presenting itself as an iPhone 14 Pro. The page immediately re-rendered in single-column mobile layout. Images grew full-width. Touch events were enabled.

---

### Tool 16 — `resize_page`

**What it is:** Resizes the browser window to match the emulated viewport.

**What it does:** Changes the physical window dimensions. Combined with `emulate`, this ensures the visual rendering matches what a mobile user would see.

**How it works internally:** CDP `Browser.setWindowBounds`.

**Input:**
```json
{ "width": 390, "height": 844 }
```

**What it told us:** Window snapped to 390×844. The screenshot at this point shows a true mobile view — hamburger menu, stacked product cards, large cart badge.

---

## Phase 6: Memory Snapshot

### Tool 17 — `take_memory_snapshot`

**What it is:** Captures a V8 heap snapshot of the current page.

**What it does:** Freezes the JavaScript heap, serialises all objects and their references to a `.heapsnapshot` file. This file can be loaded in Chrome DevTools → Memory panel for leak analysis.

**How it works internally:** CDP `HeapProfiler.takeHeapSnapshot` — this is a full, blocking heap traversal. It pauses JS execution on the page while the snapshot is taken.

**Input:**
```json
{ "filePath": "demo-shopping/memory-before-checkout.heapsnapshot" }
```

**Output:** `Heap snapshot saved to demo-shopping/memory-before-checkout.heapsnapshot`

**File size:** 6.7 MB

**What it told us:** At the point of capture (6 items in cart, mobile emulation active, several page navigations completed), the heap was 6.7 MB. For a React SPA of this complexity, this is normal. To detect a leak, you would compare this against a second snapshot taken after additional navigations and look for growing object counts in the `(closure)` or `Array` categories.

---

## Phase 7: Checkout

### Tool 18 — `fill_form` (checkout shipping info)

The checkout form at `/checkout-step-one.html` has 3 fields: First Name, Last Name, Zip/Postal Code.

**Input:**
```json
{
  "elements": [
    { "uid": "8_5", "value": "Jane" },
    { "uid": "8_6", "value": "Tester" },
    { "uid": "8_7", "value": "94107" }
  ]
}
```

**Obstacle:** `fill_form` said "Successfully filled" but the screenshot showed empty fields. This is the React controlled-input quirk: the tool sets `element.value` directly but doesn't fire the React `onChange` event, so the component's state never updates.

**Fix:** Used `evaluate_script` with the native value setter trick:

```javascript
const setNativeValue = (el, value) => {
  const setter = Object.getOwnPropertyDescriptor(
    window.HTMLInputElement.prototype, 'value'
  ).set;
  setter.call(el, value);
  el.dispatchEvent(new Event('input', { bubbles: true }));
  el.dispatchEvent(new Event('change', { bubbles: true }));
};
```

This uses React's own input tracking mechanism by replacing the value via the prototype's native setter (bypassing React's override), then dispatching a bubbling `input` event that React's synthetic event delegation picks up.

**Result:** Fields showed "Jane", "Tester", "94107" in the screenshot.

---

### Tool 19 — `press_key`

**What it is:** Presses a keyboard key or key combination.

**What it does:** Dispatches a real keydown/keypress/keyup event sequence to the focused element.

**How it works internally:** CDP `Input.dispatchKeyEvent`.

**Input (two calls):**
```json
{ "key": "Tab" }
{ "key": "Enter" }
```

**What it told us:** Tab moved focus from the zip field toward the Continue button. Enter did not submit the form (focus was still on an intermediate element). Used `evaluate_script` to click Continue instead. **Lesson:** `press_key` is most reliable for keyboard shortcuts (Ctrl+A, Escape, arrow keys) and less reliable for form submission navigation.

---

### Tool 20 — `evaluate_script` (order summary)

After reaching the Overview page, we extracted the financial totals:

```javascript
() => ({
  subtotal: document.querySelector('.summary_subtotal_label')?.textContent,
  tax: document.querySelector('.summary_tax_label')?.textContent,
  total: document.querySelector('.summary_total_label')?.textContent
})
```

**Output:**
```json
{ "subtotal": "Item total: $129.94", "tax": "Tax: $10.40", "total": "Total: $140.34" }
```

6 products × mixed prices = $129.94 subtotal. Tax rate: ~8% (San Francisco 94107 ZIP). Order total: **$140.34**.

---

## Order Confirmation

After clicking Finish, the page displayed:

> **"Thank you for your order!"**
> "Your order has been dispatched, and will arrive just as fast as the pony can get there!"

Cart badge disappeared. Session complete.

---

## Obstacles Encountered

| # | Obstacle | Root Cause | Fix |
|---|----------|-----------|-----|
| 1 | `click` on "Add to cart" didn't add item | CDP `Input.dispatchMouseEvent` doesn't trigger React synthetic `onClick` | `evaluate_script` with `element.click()` inside page context |
| 2 | `fill_form` left fields visually empty | React controlled inputs override `value` — direct assignment bypasses `onChange` | `evaluate_script` with native value setter + `input` event dispatch |
| 3 | `press_key Enter` didn't submit form | Focus was on zip field, not Continue button | `evaluate_script` to call `btn.click()` directly |
| 4 | `new_page` with `background: true` selected new tab | MCP server auto-selects newly created pages | Immediately called `select_page` to restore the intended tab |
| 5 | `take_memory_snapshot` failed — directory not found | `demo-shopping/` folder didn't exist yet | Created folder with `mkdir`, then retried |

---

## Tools NOT Triggered (and Why)

| Tool | Why not used |
|------|-------------|
| `handle_dialog` | saucedemo.com never opens browser alert/confirm/prompt dialogs. Documented for completeness. |
| `upload_file` | No file input elements exist on saucedemo. Would need a profile photo or import feature. |
| `drag` | No drag-and-drop UI on saucedemo. Would suit a kanban board or image reorder demo. |
| `type_text` | Path used `fill_form` + `evaluate_script` instead. `type_text` is for typing into a *focused* field character-by-character (simulates real keystroke timing). |
| `get_console_message` (by ID) | Console was clean throughout — no message IDs to fetch. |

---

## Files Produced

```
demo-shopping/
├── report.md                          ← findings, tool coverage, order summary
├── walkthrough.md                     ← this file
├── order-confirmation.png             ← screenshot of "Thank you for your order!"
└── memory-before-checkout.heapsnapshot ← 6.7 MB V8 heap dump (pre-checkout state)
```
