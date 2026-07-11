---
name: explainer-html
description: Build a self-contained interactive HTML explainer for a concept, system, or project — visual, clickable, plain-English + technical, zero external dependencies. Use when the user asks for an "explainer html", "interactive visual", "html app to understand X", or wants a hard concept made visual.
---

# explainer-html

One HTML file that teaches one thing well. Requested constantly — hold this quality bar
every time.

## Hard requirements
- **Single self-contained file**: inline CSS + JS, no CDNs, no fetch — must work offline
  from `file://`.
- **Actually interactive**: buttons, steppers, sliders, tabs that drive the visualization.
  Before declaring done, mentally trace every click handler — broken/no-op buttons and
  dead tabs have shipped before. If Chrome DevTools/console testing is possible, do it;
  at minimum re-read the JS for unwired handlers.
- **Three layers per concept**: plain-English analogy → technical detail → relatable
  example. Layman-first; abstract-only explanations don't land.
- **Step-through over static**: prefer "Next step" walkthroughs that animate the process
  (state highlighted at each step) over static diagrams.
- Readable typography, dark-mode friendly, generous sizing.

## Structure that works
1. Tabs or sections: one per sub-concept.
2. A "try it yourself" input where feasible (user types their own example, viz updates).
3. Optional quiz tab: MCQs with live score counter (top-right), answers + reasons revealed
   on submit.

## Delivery
- Save where the related work lives (LifeOS learning slug, project `docs/`, etc.) — keep
  all session HTMLs and MDs in ONE folder.
- `open <path>` it in the browser at the end so the user sees it immediately.
- If it will be public (GitHub Pages), include the ❤️🎈 signature footer.
