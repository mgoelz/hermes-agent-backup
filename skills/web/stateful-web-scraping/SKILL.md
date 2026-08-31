---
name: stateful-web-scraping
description: "Use when HTTP-level scraping fails on AJAX/JSF portals."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [Scraping, Playwright, Puppeteer, JSF, AJAX, Headless-Browser, Automation]
    related_skills: [blocked-page-recovery]
---

# Stateful Web Scraping (Headless Browser)

Use this when the target site is not a plain document but an **application**:
search forms that POST into a server-side session, JSF/JavaServer Faces
ViewState, ASP.NET WebForms, multi-step wizards, or content that only appears
via AJAX after a genuine UI event (popup windows, dialogs, lazy tables).

Related: `blocked-page-recovery` handles pages you cannot fetch at all (WAF,
paywall, 403). This skill is for pages you CAN fetch but whose flows do not
replay over plain HTTP.

## Decision rule: HTTP replay vs. headless browser

Try request-level scraping first ONLY if all of these hold:
- the form is a plain HTML form with no client-side state mutation
- no AJAX fires between page load and submit (check for `mojarra.ab`,
  `javax.faces.partial.ajax`, `X-Requested-With`, or React/Vue hydration)
- content is fully present in the response HTML of a GET/POST you can replay

Give request-level replay **a bounded budget: ~3-4 systematic attempts**. If
the server answers every variant with the same tiny stub response (e.g. a
200-byte `<partial-response>` containing only a new ViewState), STOP iterating
parameter combinations. That is the signature of a flow that depends on real
browser session/JS context, not on finding the magic parameter. Go headless.

## The headless-browser pattern (validated)

The following sequence worked end-to-end against a JSF/Mojarra portal where
request-level replay of the AJAX detail-click produced only ViewState stubs:

### 1. Tool choice

- **Playwright first.** Its `playwright install chromium` ships platform-native
  builds including Linux aarch64. Puppeteer's Chrome-for-Testing downloads are
  x86-64-only for the 'linux_arm' path in practice and fail with a shell-syntax
  error on ARM hosts (the binary is an x86-64 ELF).
- Launch args for root/containers: `['--no-sandbox', '--disable-dev-shm-usage']`.
- Set a real UA and `locale` (de-DE for German portals) in a fresh context.

### 2. Filling forms inside JS frameworks

Do NOT set `.value` directly — framework listeners miss the change. Use the
native prototype setter and re-fire events (this is the React/JSF-safe way):

```js
const setVal = (el, val) => {
  const proto = el.tagName === 'SELECT'
    ? window.HTMLSelectElement.prototype : window.HTMLInputElement.prototype;
  Object.getOwnPropertyDescriptor(proto, 'value').set.call(el, val);
  el.dispatchEvent(new Event('input',  { bubbles: true }));
  el.dispatchEvent(new Event('change', { bubbles: true }));
};
```

Then click the real submit button with `page.click()` (not a synthetic POST).

### 3. Reading results tables

Wait for a stable selector (`page.waitForSelector('#tbl_ergebnis')`-style)
instead of a fixed sleep, then extract rows in ONE `page.evaluate` — map each
row to a dict there, not via many round-trips.

### 4. AJAX popups opened with window.open

When a per-row button opens a POPUP WINDOW with the detail content (common on
government/legal portals), don't fight it — intercept the new page at context
level BEFORE clicking:

```js
const popupTexts = [];
ctx.on('page', async (pop) => {
  await pop.waitForLoadState('domcontentloaded');
  await pop.waitForSelector('#veroefftext', { timeout: 15000 }).catch(() => {});
  popupTexts.push(await pop.evaluate(() => document.body.innerText));
  await pop.close();
});
// then click per-row buttons; after each click, poll until popupTexts grows
```

To map popup results back to rows, collect the per-row button handles with
`page.$$(selector)` in DOM order and track their row indices from the button's
`name`/`id` attribute.

### 5. Server-side limits

Government portals often cap result lists (e.g. 1000 hits → "zu viele
Treffer"). Query in SMALL windows (single day, single state) and iterate —
never one wide query per week.

## Pitfalls

- **Don't loop on AJAX variants.** If `execute=@this`, `partial.render`,
  `behavior.event`, x/y coordinates etc. all return the same stub, the fix is
  the browser, not parameter permutation #7.
- **Reverse-engineering the site's own JS is allowed and cheap**: fetch the
  portal's `.js` files (e.g. `inso.js`) and read them — they reveal exactly
  what the UI does (`popupWindow.location.href = "text.xhtml?..."` saved an
  hour of guessing).
- **JSESSIONID in URL path** (`;jsessionid=...`) is normal for JSF portals;
  requests-based scrapers must keep the full URL, browsers handle it alone.
- **Parse with layered regex strategies** for messy public-sector text: try a
  specific pattern first, fall back to broader ones, and validate against 3+
  real formatting variants (formulations differ per court/office).
- **Deduplicate persisted artifacts**: when a pipeline writes
  `foo.json` + `foo_dossiers.json` into one dir, glob filters must exclude the
  derived files or the next run picks up the wrong input.

## ARM hosts (aarch64)

Check `uname -m` before installing a browser. On aarch64 use Playwright only:
Puppeteer downloads an x86-64 Chrome binary that fails at launch with
`Syntax error: Unterminated quoted string` (the shell cannot exec an x86-64
ELF). Playwright's `npx playwright install chromium` fetches a native aarch64
build. Install time is several minutes — run it in the background with
notification, not foreground (600 s cap can be too short).

## Backfilling history (resumable bulk runs)

Stateful portals often serve HISTORY, not just today's data — probe the depth
once (e.g. a 2024 date and a ~5-years-ago date) before assuming you must
collect-only-forward. A historical archive turns a daily monitor into a
sellable dataset (backfill as one-time product + live feed as subscription).

Pattern for multi-day/week bulk collection:

- **One file per unit of work** (`data/backfill/<unit>.json`), written
  immediately after each unit completes. Skip units whose file exists → the
  run is RESUMABLE: on crash or rate-limit, just relaunch the same command.
- **Reuse one browser across units**; open a fresh context per unit (cheap,
  avoids state bleed). On repeated failure, close and relaunch the browser —
  corrupted browser state, not the target, is the usual cause after hour one.
- **3 retries per unit**, then log and skip (a gap beats a deadlock).
- **Prioritize expensive fetches**: if detail views cost seconds each, fetch
  details only for priority-classified rows (regex on names) with a small cap
  per unit; you can always deep-fetch later since files are the index.
- **Log to a file with progress `[n/total]`** so a background run is
  observable via tail; politeness sleep (1.5–3 s + jitter) between units.
- Measure per-unit cost first (e.g. ~4–5 s/day) and multiply before promising
  a completion time to the user; run via terminal(background=true, notify=true).

## Session-specific example

See `references/jsf-portal-insolvenz.md` for a full worked example: scraping
Germany's official insolvency announcements portal (search form → results
table → per-row publication-text popups), including the working Playwright
script structure, data volumes, and the classification/digest pipeline built
on top.