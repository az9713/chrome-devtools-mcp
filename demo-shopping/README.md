# demo-shopping

An agentic shopping workflow on saucedemo.com — our second chrome-devtools-mcp demo.

---

## Contents

### [`report.md`](report.md)
Findings, tool coverage table, order summary ($140.34), and recommendations.

### [`walkthrough.md`](walkthrough.md)
Full annotated walkthrough: every tool, every input/output, every obstacle and fix.

### [`order-confirmation.png`](order-confirmation.png)
Screenshot of the "Thank you for your order!" confirmation page.

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

## Suggested Reading Order

1. `report.md` — see what happened and what was found
2. `walkthrough.md` — understand exactly how it was done
