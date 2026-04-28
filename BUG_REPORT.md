# iPadOS Ctrl+C keyCode=13 Bug Report (WebKit / UIKit)

## Title
iPadOS Safari reports keyCode=13 instead of 67 for Ctrl+C keydown with hardware keyboard

## Product / Component
WebKit / UI Events

## Environment
- iPad with hardware keyboard (Magic Keyboard or any external keyboard)
- iPadOS (tested on Version 26.4 / Safari 26.4)
- Safari (bundled with iPadOS)

## Steps to Reproduce
1. Connect a hardware keyboard to the iPad
2. Open `webkit-ctrl-c-test.html` in Safari
3. Focus the textarea
4. Press and hold the **Ctrl** key, then press **C**
5. Observe the keydown event logged below the textarea

## Actual Result
```
key="c", code="KeyC", keyCode=13, which=13, ctrlKey=true
```
`keyCode` is **13** (Enter/Return's keyCode)

## Expected Result
```
key="c", code="KeyC", keyCode=67, which=67, ctrlKey=true
```
`keyCode` should be **67** (C's keyCode), matching macOS Safari behavior.

## Comparison (same iPad + same hardware keyboard)

| Key combination | key | keyCode | expected keyCode | Status |
|---|---|---|---|---|
| Ctrl+A | "a" | 65 | 65 | ✅ Correct |
| **Ctrl+C** | **"c"** | **13** | **67** | **❌ Bug** |
| Ctrl+D | "d" | 68 | 68 | ✅ Correct |
| Ctrl+E | "e" | 69 | 69 | ✅ Correct |
| Enter (no Ctrl) | "Enter" | 13 | 13 | ✅ Correct (unaffected) |
| Ctrl+Enter | "Enter" | 13 | 13 | ✅ Correct |

## Root Cause Analysis (by layer)

### Layer 1 – Hardware / Low-level input
✅ **Normal.** `e.key = "c"` and `e.code = "KeyC"` are correct, confirming that
the physical keystroke is properly decoded at the hardware layer.

### Layer 2 – UIKit (iPadOS input processing layer)
❌ **Primary suspicion.** On iPadOS, Ctrl+C is a system shortcut for "Copy".
UIKit intercepts the combination via `UIKeyCommand` / `UIResponder` before it
reaches WebKit. The key observation:

| Property | Value | Correct? |
|---|---|---|
| `e.key` | `"c"` | ✅ correct (modern path preserved) |
| `e.keyCode` | `13` | ❌ **wrong** (legacy path corrupted) |

This split suggests that UIKit's interception corrupts the legacy keyCode
mapping while preserving the modern `key` property. The fact that **only
Ctrl+C** (the one combination that is a system shortcut) is affected is a
strong signal that UIKit's shortcut handling is the trigger.

### Layer 3 – WebKit (DOM event generation)
⚠️ **Not entirely innocent.** WebKit constructs the `KeyboardEvent` object and
is ultimately responsible for filling `keyCode`. Even if UIKit passes
corrupted data downstream, WebKit chooses the fallback mapping that maps
the event to `keyCode=13` (Enter) instead of `keyCode=67` (C).

The cleanest description:

> **UIKit's system shortcut interception corrupts the legacy keyCode mapping,
> and WebKit's compatibility layer does not recover from this corruption,
> falling back to keyCode=13.**

## Suggested verification experiments

To further isolate the responsible layer:

1. **`preventDefault()` on keydown** — If intercepting the event before
   UIKit processes it helps, this would confirm UIKit involvement.
2. **`contenteditable` div instead of `<textarea>`** — Rules out any
   textarea-specific code path.
3. **WKWebView (native app)** — If the bug reproduces in a WKWebView-based
   app, the bug is definitely in UIKit → WebKit plumbing, not Safari-specific.
4. **Non-WebKit browser on iPadOS** — Not possible due to Apple's policy
   (all browsers on iPadOS are WebKit-based), but worth noting.

## Impact
Applications relying on `keyCode` for keyboard handling — terminal emulators
(GoTTY, ttyd, xterm.js), code editors (CodeMirror, Ace), games, and legacy
web apps — misinterpret Ctrl+C as Enter (`\r`), breaking SIGINT / copy /
cancel functionality.

## References
- Reproduction repo: https://github.com/ShoichiTect/webkit-ctrl-c-bug
- HTML test page: `webkit-ctrl-c-test.html` in the repo above
- GoTTY fix documenting the same issue: https://github.com/ShoichiTect/gotty
