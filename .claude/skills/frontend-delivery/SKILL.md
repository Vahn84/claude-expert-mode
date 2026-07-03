---
name: frontend-delivery
description: Use when working on frontend performance, UI architecture, accessibility, a component library, or an SPA that "feels slow" — before proposing fixes. Measure the real user-perceived metric before changing anything; fix systemic UI defects at the primitive level; gate the fix so it doesn't regrow.
---

# Frontend: Measure the Right Number, Fix at the Primitive, Gate the Regrowth

Frontend work has three reliable traps: optimizing the wrong metric, fixing symptoms scattered across components, and landing a fix with no gate so it decays in a year. Run these before proposing changes.

## 1. Measure the user-perceived metric first — "feels slow" is not a number
"Feels slow" splits into opposite problems with opposite fixes:
- **Load** (LCP/first paint) → bundle bloat, fetch waterfalls, no code-splitting.
- **Interaction** (INP) → re-render storms, heavy handlers, over-broad context.
Get **field RUM** (real-user Core Web Vitals: LCP/INP/CLS), not just a lab Lighthouse run, plus a bundle analysis (source-map-explorer). Don't optimize load if the pain is interaction. Attribute before you touch.

## 2. Accessibility failures are systemic — fix the primitives, not the pages
WCAG failures in a grown app are a handful of defects repeated everywhere, not scattered one-offs. Get the **auditor's exact criteria** (or run axe-core for the machine-detectable ~40%, plus a manual keyboard + screen-reader pass for the rest). Then fix at the leverage point:
- Contrast → fix once at the design-token level.
- `<div onClick>` soup → replace with a small set of accessible primitives.
- Focus management/traps, focus indicators → add to the shared dialog/menu components.
- Dynamic updates unannounced → `aria-live` in the shared patterns.
"Every team rolled their own button" is usually the *root cause* of both the a11y failure and the inconsistency — one accessible component library (adopt headless/a11y-correct primitives like Radix or React Aria; don't hand-build) clears it in bulk.

## 3. Server state does not belong in component state / context
Ad-hoc `fetch` in components + deep context nesting is the usual source of both fetch chaos and re-render cascades. Pulling server state into a query library (TanStack Query or equivalent) often fixes both at once. Name this when you see 20+ contexts or fetch-in-render.

## 4. Gate it or it regrows — this is the actual deliverable
A fix without a gate decays. Land these as PR gates, not one-time cleanups:
- axe / Lighthouse-CI as a merge check,
- `eslint-plugin-jsx-a11y` failing the build,
- a bundle-size / performance budget.
The gate is what makes the fix permanent; without it you are re-doing this next year.

## 5. Scope honestly
Perf + a11y + architecture rarely all fit in one budget. State plainly which is *guaranteed* (usually the contract-gated a11y work), which gets *measured high-leverage wins* (perf), and which you only *seed* (the library + gates; the migration off legacy state is quarters). Don't imply the whole thing fits when it doesn't.

## Why this exists

Handover finding: the domain judgment is strong when engaged; this gate ensures it fires in order (measure → primitive-level fix → gate) rather than jumping to fixes, and that the CI gate — the part that makes it durable — isn't left as an afterthought.
