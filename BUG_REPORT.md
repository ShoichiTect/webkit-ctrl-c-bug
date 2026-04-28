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

### Layer 2 – UIKit / UIKeyCommand (system shortcut interception)
❌ **Primary cause.** On iPadOS, Ctrl+C is a system shortcut for Copy.
UIKit intercepts it at the **system level** via `UIKeyCommand` / `UIResponder`
— NOT at the DOM event level. Verified evidence:

| Experiment | Result | Implication |
|---|---|---|
| `preventDefault()` on keydown | **keyCode stays 13** | Corruption happens before JS can intervene |
| `preventDefault()` ON → `[COPY EVENT]` | **Does NOT fire** | UIKit consumes Copy at system level, no DOM copy event |
| `preventDefault()` OFF → `[COPY EVENT]` | **Does NOT fire** | Same — system-level copy, not DOM-level |
| `preventDefault()` OFF → `beforeinput` | **Fires with CR** | WebKit treats corrupted event as text input (Enter/CR) |

Key observation:

| Property | Value | Correct? | Path |
|---|---|---|---|
| `e.key` | `"c"` | ✅ correct | modern API — takes a different code path |
| `e.code` | `"KeyC"` | ✅ correct | physical key location — separate path |
| `e.keyCode` | `13` | ❌ **wrong** | legacy compat — **corrupted by UIKit** |
| DOM `copy` event | **never fires** | N/A | UIKit does system pasteboard, not DOM |

### Layer 3 – WebKit (DOM event construction)
⚠️ **Contributing factor.** WebKit constructs the `KeyboardEvent` object and
fills `keyCode` with the value received from the system layer. The corruption
happens before WebKit receives the event, so WebKit cannot fix it.

However, WebKit **chooses the mapping** through its legacy keyCode
compatibility code. Even with a corrupted system value, WebKit could
theoretically recover the correct keyCode from `e.key` (which is correct).
The fact that it does not is a secondary bug in WebKit's compatibility layer.

### Visual data flow

```
[Hardware Keyboard]
    ↓
[UIKit / UIKeyCommand]
    ├── Ctrl+C intercepted as system Copy
    ├── keyCode corrupted → 13 (Enter)  ← ❌ PRIMARY BUG
    └── e.key = "c" preserved correctly
    ↓  (no DOM copy event generated)
[WebKit]
    ├── Receives corrupted keyCode
    ├── Could recover from e.key but doesn't  ← ⚠️ SECONDARY BUG
    └── Faithfully reflects keyCode=13 in KeyboardEvent
    ↓
[keydown]  e.key="c"  keyCode=13  ctrlKey=true
    ↓
[keypress] keyCode=13 (CR character)
    ↓
[beforeinput → input]  ← text input processing
```

### Summary

> **UIKit intercepts Ctrl+C at the system level (UIKeyCommand), corrupting
> keyCode to 13. WebKit receives the corrupted value and passes it through
> to the DOM without correction, despite having the correct `e.key` available
> for recovery. No DOM `copy` event is generated because the copy operation
> is handled entirely at the system level.**

## Verification experiments (results)

| # | Test | Result | Conclusion |
|---|---|---|---|
| 1 | `preventDefault()` on keydown | **keyCode remains 13** | Corruption predates JS event handling |
| 2 | DOM `copy` event listener | **Never fires** | Copy is system-level, not DOM-level |
| 3 | `preventDefault()` ON → copy event | **Never fires** | Same as above |
| 4 | `contenteditable` | (not tested) | Would confirm element-type independence |
| 5 | WKWebView | (not tested) | Would isolate from Safari-specific code |

Experiments 1-3 were performed on iPadOS 26.4 / Safari 26.4 with Magic
Keyboard. Raw event log data is available in the repository.

## Impact
Applications relying on `keyCode` for keyboard handling — terminal emulators
(GoTTY, ttyd, xterm.js), code editors (CodeMirror, Ace), games, and legacy
web apps — misinterpret Ctrl+C as Enter (`\r`), breaking SIGINT / copy /
cancel functionality.

## References
- Reproduction repo: https://github.com/ShoichiTect/webkit-ctrl-c-bug
- HTML test page: `webkit-ctrl-c-test.html` in the repo above
- GoTTY fix documenting the same issue: https://github.com/ShoichiTect/gotty
