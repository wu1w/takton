# Bug audit notes — v0.2.6

## Fixed in this release pass
- Linux desktop screenshot: scrot path now uses `_finalize_screenshot` (no accidental base64-only + deleted tmp path)
- Windows adapter: bare `except:` → `except Exception:`
- takton-code TUI: lazy `__init__` so importing vim_ux/renderer does not require `rich` at collection time

## Verified green
- Backend focused suites: L1/L2/L3/L4/tool policy/loop stability/unified tools — 46 passed
- takton-code tests: **107 passed** (PYTHONPATH=takton-code/src)

## Known / deferred (non-blocking)
- cron-hook SubAgent path still TODO
- cluster cancel TODO
- desktop DB permission clear TODO
- Physical seat0 currently GDM greeter (no wuyw GUI login); E2E validated on real :0 greeter + Xvfb
- vendor/takton-code binary still not in git (by design); build copies onefile at pack time
- Double normalize: loop also normalizes tool results after registry (harmless)

## Embedded takton-code
- Source: `takton-code/` (version field still 0.1.0 package-local; product ships as Takton 0.2.6 component)
- Vendor binary: `vendor/takton-code/README.md` only
