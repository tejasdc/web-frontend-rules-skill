---
name: web-frontend-rules
description: Use when building or fixing web UI Tejas has already ruled on — a shared nav, footer, header or any component used on several pages, "the nav moves between pages", "it jumps when I navigate", "these should line up exactly", "pixel-identical", "alignment is off by a few pixels", React or Next.js data fetching and memoized props, and choosing browser-testing tools on the server versus the Mac (Playwright on Linux, webkit-pilot and the iOS Simulator only on the laptop). Pairs with `browser-verification` and `vercel-react-best-practices`.
---

# Web front-end rules

Moved verbatim from the global instruction file on 2026-09-29.

## Shared UI components must be typographically self-contained

Any component used across multiple pages (site nav, footer, shared chrome, injected islands) must set its own `font-size`, `line-height`, and use absolute-unit `padding` — not `rem`/`em` that inherit from the host page's `<html>`/`<body>`. Different pages routinely set different root font-sizes (16px, 17px, 18px) and different `line-height` values, so a component using `padding: 1.15rem` renders at 18.4px on one page and 19.55px on another, and its child anchors inherit different line-heights that change computed height. From the outside it looks like the component "moves" when navigating between pages. The user calls this a bug even when the source CSS is the same.

Concretely, in a shared component's own `<style>`:
- Use `px` (or `vw`/`svh` for viewport-relative), not `rem`/`em`, for padding/margin/gaps.
- Set `line-height` explicitly on the component's root and on any text children (usually `1` for compact chrome, an explicit ratio like `1.4` for prose).
- Optionally set `font-size` explicitly too if the component's typographic scale needs to be independent of host.

This is the same principle as `box-sizing: border-box` on JS-injected elements — the component must not depend on the host page's cascade for its own geometry to be correct.

## When two surfaces must be pixel-identical, MEASURE — don't eyeball

"Same component, both pages use it, should be identical" is a wish, not a proof. Astro scoped CSS, host-page font-size, host-page line-height, and inherited border-box can all silently make two instances of the same component render at different pixel positions. Before declaring identical alignment done:

1. Load both live URLs in a real browser.
2. Run `element.getBoundingClientRect()` on the elements that must match (nav items, first-line baselines, whatever's supposed to be aligned).
3. Compare `top`, `left`, `height`, `width` numerically. Assert zero pixel difference (or a specific tolerance the user has approved).
4. Also print `getComputedStyle(el).padding` / `line-height` / `font-size` on both — if the numbers differ, the render will drift even when the source looks the same.

Screenshots and reading source CSS are not sufficient. The compiled+inherited values are what matter, and only measurement shows them.

## A row of controls in shared chrome never shrinks or wraps; the text gives way

In any header, heading or toolbar with a title beside controls, the control group is `flex: none` with
`flex-wrap: nowrap`, and the title takes what is left (`min-width: 0`, then ellipsis or wrap). A
control group left to shrink wraps its own buttons the moment a title is long, and at phone width a
close mark lands under its neighbour. Opening a native `<dialog>` with `showModal()` also focuses its
first control, which a phone then draws with a focus ring; name the element to focus, or focus the
window or its title, so nothing arrives ringed. Measure the shared chrome at 390 points with the
longest real title before shipping it, not only the surface you changed.

> "The file header is again, like taking a ball of the space, and look at how the alignment is broken
> for the back button. Like, why, why are we building things like this?" (his report, 2026-09-30; the rule is an agent choice from it, [decision: window-controls-on-one-line];
> thnkr.ing report f25b02ab: the file window's heading was 117 points tall with the close mark under
> the bug mark, its details 107 more, the text starting 245 points down. Fixed in thnkr.ing 78a3e85;
> its look tool now has a `file-open` state for this window.)

## Browser verification — choose tools by execution host

**Linux / remote-box:** use standalone Playwright tests. Select engine and viewport coverage by affected behavior and repository policy; `browser-verification` owns that judgment, screenshot inspection, and evidence. Use `local-test` for fixture ownership and native parallel execution. Native iPhone Safari verification is not required. Do not require `webkit-pilot`, Xcode, or iOS Simulator, or treat their absence as a release blocker on Linux.

**Laptop / macOS:** `webkit-pilot` (native WKWebView) and iOS Simulator are optional laptop-only tools for Safari-focused testing when the task calls for it and they are available. They are not prerequisites for ordinary web changes. Native iPhone testing enters scope only when the user explicitly requests it.

**Interactive browsing:** use an available tool on the current host, such as `agent-browser` or a connected browser extension. Chrome extension tools are optional; when using them, get tab context first. Reusable UI verification belongs in standalone tests rather than a mandatory MCP/extension workflow.

Report the engines and viewport sizes actually exercised, and investigate engine-specific failures in that engine. Call Linux coverage “WebKit on Linux”; do not claim it is native Safari. Missing native Apple tooling is outside the normal Linux verification scope, so do not repeatedly present it as unfinished work. Distinguish a missing browser executable from a localhost permission error or a failing test.

Source: Tejas's 2026-09-07 browser-verification policy clarification; the former laptop-specific rule had incorrectly become a global iPhone-testing requirement.


## React and Next.js

For any React or Next.js work — new components, hooks, data fetching, state management, subscription plumbing, or performance/hang investigation — load `vercel-react-best-practices`. Its 70-rule catalog covers render, effect, subscription, bundle and server patterns. Two class-level architectural principles the current catalog does NOT state should be applied alongside it: (1) **data-layer separation** — any fetch returning identifiable data (id/url/path/hash-keyed) goes through a shared cache module; components observe caches, they do not own network calls except for one-shot user-gesture mutations. (2) **identity preservation across renders** — non-primitive props passed to memoized children or third-party runtimes (assistant-ui, tanstack, radix, etc.) must preserve reference identity when the underlying data hasn't changed; use keyed conversion caches or structural sharing, not fresh transforms per render. Per-project `AGENTS.md` names the local cache-layer module and the off-limits HTTP verbs. Source: 2026-09-16 Safari PWA hang investigation in Thinkering.

## Gestures are not events, and portals bubble in React

Two rules from one night on his iPhone (thnkr.ing thread reply box, 2026-10-07):

1. **Close, dismiss or minimise only on his gesture, never on an event the page can raise by itself.** A `scroll` event fires when the keyboard dismisses and the viewport grows, when a list re-lays out, when a notice above the scroller leaves; none of those is a finger. Read the gesture: `touchstart` then `touchmove` past a threshold over the thing he scrolls, or `wheel`; a tap outside is `pointerdown`/`pointerup` within a few pixels, judged with `composedPath()`. A box that minimised on the conversation's scroll closed the moment he tapped its own To chooser, because the chooser dismissed the keyboard.
2. **A React portal is inside its parent's synthetic event tree.** `onPointerDown` on a tray fires for a tap on a listbox that `createPortal` drew on the body, so a drag handler there captured the pointer and the option never received its click. Decide "is this my surface" by DOM containment (`currentTarget.contains(target)`), not by React ancestry; and when deciding "outside the box" for a portaled list, tie the list to its trigger (a data attribute both carry) rather than assuming containment.
