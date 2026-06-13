---
name: chrome-cdp
description: Interact with local Chrome browser session (only on explicit user approval after being asked to inspect, debug, or interact with a page open in Chrome)
---

# Chrome CDP

Lightweight Chrome DevTools Protocol CLI. Connects directly via WebSocket — no Puppeteer, works with 100+ tabs, instant connection.

## Prerequisites

- Chrome (or Chromium, Brave, Edge, Vivaldi) with remote debugging enabled: open `chrome://inspect/#remote-debugging` and toggle the switch
- Node.js 22+ (uses built-in WebSocket)
- If your browser's `DevToolsActivePort` is in a non-standard location, set `CDP_PORT_FILE` to its full path
- If you already know the browser-level WebSocket endpoint, set `CDP_WS_URL=ws://host:port/devtools/browser/<id>`

## Commands

All commands use `scripts/cdp.mjs`. The `<target>` is a **unique** targetId prefix from `list`; copy the full prefix shown in the `list` output (for example `6BE827FA`). The CLI rejects ambiguous prefixes.

### Diagnose Chrome CDP availability

```bash
scripts/cdp.mjs doctor
```

Run this before browser work if the current Chrome CDP state is unknown. If it reports `No DevToolsActivePort found`, open `chrome://inspect/#remote-debugging` in the real Chrome app, enable remote debugging for that browser session, then rerun `doctor` and `list`.

If `doctor` reports `Local debug port 127.0.0.1:9222: http-no-json (HTTP 404)`, Chrome is in the Chrome 144+ auto-connect UI mode, but this is **not** a usable raw-CDP endpoint for `cdp.mjs`: `/json/version` returns 404 and no `DevToolsActivePort` file exists. Do not run `list`, `open`, or page commands in that state. For this skill, raw CDP needs either a `DevToolsActivePort` file or an explicit `CDP_WS_URL=ws://host:port/devtools/browser/<id>`.

### List open pages

```bash
scripts/cdp.mjs list
```

### Take a screenshot

```bash
scripts/cdp.mjs shot <target> [file]    # default: screenshot-<target>.png in runtime dir
```

Captures the **viewport only**. Scroll first with `eval` if you need content below the fold. Output includes the page's DPR and coordinate conversion hint (see **Coordinates** below).

### Accessibility tree snapshot

```bash
scripts/cdp.mjs snap <target>
```

### Evaluate JavaScript

```bash
scripts/cdp.mjs eval <target> <expr>
```

> **Watch out:** avoid index-based selection (`querySelectorAll(...)[i]`) across multiple `eval` calls when the DOM can change between them (e.g. after clicking Ignore, card indices shift). Collect all data in one `eval` or use stable selectors.

### Other commands

```bash
scripts/cdp.mjs html    <target> [selector]   # full page or element HTML
scripts/cdp.mjs nav     <target> <url>         # navigate and wait for load
scripts/cdp.mjs net     <target>               # resource timing entries
scripts/cdp.mjs click   <target> <selector>    # click element by CSS selector
scripts/cdp.mjs clickxy <target> <x> <y>       # click at CSS pixel coords
scripts/cdp.mjs type    <target> <text>         # Input.insertText at current focus; works in cross-origin iframes unlike eval
scripts/cdp.mjs loadall <target> <selector> [ms]  # click "load more" until gone (default 1500ms between clicks)
scripts/cdp.mjs evalraw <target> <method> [json]  # raw CDP command passthrough
scripts/cdp.mjs open    [url]                  # open new tab (reuses the hub; no new prompt)
scripts/cdp.mjs stop                           # stop the hub (ends the shared session)
```

## Coordinates

`shot` saves an image at native resolution: image pixels = CSS pixels × DPR. CDP Input events (`clickxy` etc.) take **CSS pixels**.

```
CSS px = screenshot image px / DPR
```

`shot` prints the DPR for the current page. Typical Retina (DPR=2): divide screenshot coords by 2.

## Tips

- Prefer `snap` over `html` for page structure (the snapshot is already compacted).
- Use `type` (not eval) to enter text in cross-origin iframes — `click`/`clickxy` to focus first, then `type`.
- Chrome 144+ asks you to approve each remote-debugging *connection*. A single background hub holds one connection open and runs every command (`list`, `doctor`, and all page commands across every tab) over it, so you approve **once per session**, not once per tab or per `list`/`doctor` call. The hub auto-exits after 8 hours of inactivity by default; override with `CDP_IDLE_TIMEOUT_MS`.
- That one approval lasts until the hub exits (idle timeout, `scripts/cdp.mjs stop`, or Chrome closing). Avoid `scripts/cdp.mjs stop` until the browser work is done, so you don't re-trigger the prompt.
- For tasks that need the user's logged-in Chrome session but should not disturb the main window, first run `doctor`/`list`, then create or reuse a separate normal Chrome window and operate only on targets from that window. Do not launch an isolated `--user-data-dir` profile for this case because it will not share the user's session.
