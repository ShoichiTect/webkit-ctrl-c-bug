# WebKit Bug Report

## Title
iPadOS Safari reports keyCode=13 instead of 67 for Ctrl+C keydown with hardware keyboard

## Product / Component
WebKit / UI Events

## Environment
- iPad with hardware keyboard (Magic Keyboard or any external keyboard)
- iPadOS (tested on Version 26.4)
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

## Comparison (same hardware keyboard, same iPad)

| Key combination | key | keyCode | expected keyCode | Status |
|---|---|---|---|---|
| Ctrl+A | "a" | 65 | 65 | ✅ Correct |
| **Ctrl+C** | **"c"** | **13** | **67** | **❌ Bug** |
| Ctrl+D | "d" | 68 | 68 | ✅ Correct |
| Ctrl+E | "e" | 69 | 69 | ✅ Correct |
| Enter (no Ctrl) | "Enter" | 13 | 13 | ✅ Correct (unaffected) |
| Ctrl+Enter | "Enter" | 13 | 13 | ✅ Correct |

## Notes
- `e.key` (modern W3C spec) is reported correctly as `"c"`
- Only the deprecated `e.keyCode` / `e.which` compatibility property is wrong
- This suggests a bug in WebKit's legacy keyCode mapping layer on iOS
- macOS Safari is NOT affected (returns keyCode=67 correctly)
- Only Ctrl+C is affected, likely because iOS intercepts it as a system "Copy" shortcut

## Impact
Applications using `keyCode` for keyboard handling (e.g. terminal emulators, code editors, games) interpret Ctrl+C as Enter, breaking Ctrl+C/SIGINT functionality.
