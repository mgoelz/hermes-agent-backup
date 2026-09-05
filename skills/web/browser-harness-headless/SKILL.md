---
name: browser-harness-headless
description: Use when browser_exec fails with chrome-not-running.
---

# browser-harness on headless Linux

The `browser_exec` tool drives a real browser through the browser-harness daemon, which attaches to a Chromium-family browser over CDP. On a headless server **no browser is running by default**, so every `browser_exec` call fails with:

```
browser-harness: daemon default didn't come up -- check ~/.config/browser-harness/tmp/bu-default.log
# log says: fatal: chrome-not-running: no supported Chromium-family browser is running
```

This is the normal state, not a breakage. Fix: start a Chromium yourself before the first browser_exec call.

## Working startup recipe

1. Find a usable Chromium (Playwright's cache usually has one):
   ```bash
   ls ~/.cache/ms-playwright/            # e.g. chromium-1234
   find ~/.cache/ms-playwright -name chrome -path '*chrome-linux*' | head -1
   ```
   System chrome/chromium may not exist (`which google-chrome chromium` empty) — the Playwright binary works fine.

2. Start it in the background with `--user-data-dir` set to `~/.config/chromium`.
   **This is the critical flag.** The harness daemon discovers the CDP endpoint by reading `DevToolsActivePort` inside its well-known profile dirs (`~/.config/google-chrome`, `~/.config/chromium`, …). A random user-data-dir (e.g. `/tmp/chrome-x`) gives a working CDP port but the daemon still reports *chrome-not-running*, because its discovery loop never looks there.

   Via the terminal tool (use background=true, never shell `&`/nohup wrappers):
   ```bash
   ~/.cache/ms-playwright/chromium-*/chrome-linux/chrome \
     --headless=new --remote-debugging-port=9222 --no-sandbox --disable-gpu \
     --user-data-dir=$HOME/.config/chromium about:blank
   ```

3. Verify readiness in a separate call (don't blind-wait):
   ```bash
   cat ~/.config/chromium/DevToolsActivePort   # two lines: port + ws path
   curl -s http://127.0.0.1:9222/json/version   # returns Browser/Protocol JSON
   ```

4. Now `browser_exec` works normally: `new_tab(url)` first, then `wait_for_load()`, `page_info()`, `js()`, `click_at_xy()`, `capture_screenshot()`.

## Diagnostics

- `browser-harness --doctor` (binary under `~/.cache/uv/archive-v0/*/bin/browser-harness`) shows chrome running / daemon alive / connections. "chrome running" plus "daemon not alive" usually means the profile-dir mismatch above.
- The daemon's own log: `~/.config/browser-harness/tmp/bu-default.log`.
- The harness source (profile list, discovery loop) lives at
  `~/.cache/uv/archive-v0/*/lib/python3.11/site-packages/browser_harness/daemon.py` — `PROFILES` shows every directory it scans.

## Pitfalls

- Chrome must keep running for the whole browser session. If started via the terminal tool with background=true it survives across calls; a foreground launch dies when the call returns.
- Long-lived Chrome processes from earlier sessions may already hold port 9222 or lock the profile dir — kill the stale one (`process` tool) before relaunching with the same user-data-dir.
- `--no-sandbox` is required when running as a non-privileged user on servers without user-namespace sandboxing.
- Expect harness stderr noise like `Chrome is asking "Allow remote debugging?"` on headless=new — harmless; calls still succeed.
