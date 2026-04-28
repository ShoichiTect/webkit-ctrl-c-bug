# iPadOS Safari Ctrl+C keyCode=13 Bug

**Ctrl+C on iPad hardware keyboard sends `keyCode=13` (Enter) instead of `keyCode=67` (C).**

## Quick facts

| Property | Value | Correct? |
|---|---|---|
| `e.key` | `"c"` | ✅ |
| `e.code` | `"KeyC"` | ✅ |
| `e.keyCode` | `13` | ❌ should be `67` |
| `e.ctrlKey` | `true` | ✅ |

Only Ctrl+C is affected (Ctrl+A, Ctrl+D, etc. work fine). macOS Safari works
correctly. Root cause is likely **UIKit intercepting Ctrl+C as system Copy**,
with WebKit's legacy `keyCode` mapping failing to recover.

## Contents

- `webkit-ctrl-c-test.html` — standalone test page for reproduction
- `BUG_REPORT.md` — detailed bug report for WebKit Bugzilla

## Links

- GitHub: https://github.com/ShoichiTect/webkit-ctrl-c-bug
- WebKit Bugzilla: https://bugs.webkit.org/ (Product: WebKit, Component: UI Events)
