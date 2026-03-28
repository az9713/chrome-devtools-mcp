# demo-shopping

An agentic shopping workflow on saucedemo.com — our second chrome-devtools-mcp demo.

---

## Contents

### [`report.md`](report.md)
Findings, tool coverage table, order summary ($140.34), and recommendations.

### [`walkthrough.md`](walkthrough.md)
Full annotated walkthrough: every tool, every input/output, every obstacle and fix.

### [`screenshots/`](screenshots/)
10 screenshots covering every step of the workflow:

| File | Step |
|------|------|
| [`01-login.png`](screenshots/01-login.png) | Login page — `new_page` opened saucedemo.com |
| [`02-login-filled.png`](screenshots/02-login-filled.png) | Credentials filled — `fill_form` in action |
| [`03-product-catalog.png`](screenshots/03-product-catalog.png) | Product catalog — logged in, 6 items visible |
| [`04-product-detail.png`](screenshots/04-product-detail.png) | Fleece Jacket detail — `navigate_page` deep link |
| [`05-catalog-cart-full.png`](screenshots/05-catalog-cart-full.png) | All 6 items added — cart badge shows 6 |
| [`06-mobile-view.png`](screenshots/06-mobile-view.png) | iPhone 14 Pro layout — `emulate` + `resize_page` |
| [`07-cart.png`](screenshots/07-cart.png) | Cart page — all items, $129.94 subtotal |
| [`08-checkout-info.png`](screenshots/08-checkout-info.png) | Checkout step 1 — shipping form filled with `fill_form` |
| [`09-checkout-overview.png`](screenshots/09-checkout-overview.png) | Checkout step 2 — order summary, $140.34 total |
| [`10-order-confirmation.png`](screenshots/10-order-confirmation.png) | "Thank you for your order!" — purchase complete |

### [`memory-before-checkout.heapsnapshot`](memory-before-checkout.heapsnapshot)
6.7 MB V8 heap dump captured mid-session (6 items in cart, mobile emulation active).
Open in Chrome DevTools → Memory panel to explore.

---

## What This Demo Covers

| Aspect | Detail |
|--------|--------|
| Target | https://www.saucedemo.com |
| Scenario | Full e-commerce flow: login → browse → cart → checkout → confirmation |
| Tools used | 18 of 29 (the 18 not covered in the CNN health-check demo) |
| Key finding | Broken crash-analytics integration — every telemetry ping fails 401 |
| Key technique | `evaluate_script` as escape hatch for React synthetic event system |

---

## Trigger prompt

Copy this into Claude Code (with chrome-devtools-mcp configured) to run this demo yourself:

> "Using chrome-devtools-mcp, act as an autonomous shopping agent on https://www.saucedemo.com. Log in using fill_form with username standard_user and password secret_sauce. Browse the product catalog and hover over the Sauce Labs Fleece Jacket. Navigate to its detail page. Open the Sauce Labs Backpack detail page in a second background tab for comparison, then close that tab. Add all 6 products to the cart. Switch to iPhone 14 Pro mobile emulation (390×844, 3× DPR, touch enabled) and take a screenshot showing the mobile layout. Take a heap memory snapshot and save it to demo-shopping/memory-before-checkout.heapsnapshot. Navigate to the cart, proceed to checkout, fill the shipping form with first name Jane, last name Tester, zip 94107, and complete the order. Report any broken third-party integrations discovered in the network requests."

---

## Suggested Reading Order

1. `report.md` — see what happened and what was found
2. `walkthrough.md` — understand exactly how it was done
