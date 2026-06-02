# Implementation Plan: `terminal.screenshot` Command

## Overview

Add a production-ready `terminal.screenshot` v2 socket command and a `cmux terminal screenshot` CLI subcommand, mirroring the existing `browser.screenshot` pattern. The core IOSurface capture primitive already exists (DEBUG-only); this plan promotes it to production and wraps it in the standard screenshot interface.

## Design Decisions

- **Command name:** `terminal.screenshot` (parallel to `browser.screenshot`)
- **No pixel diff:** Just capture and return the PNG. The debug `panelSnapshot` command retains pixel-diff support separately.
- **Debug commands preserved:** Existing `panelSnapshot` / `debug.panel_snapshot` commands remain as-is (DEBUG-only, with pixel diff).

## Changes by File

### 1. `Sources/GhosttyTerminalView.swift` -- Promote IOSurface capture to production

**What:** Extract `copyIOSurfaceCGImage()` from the `#if DEBUG` block so it's available in all builds.

- **New method** `copyIOSurfaceCGImage() -> CGImage?` on `GhosttySurfaceScrollView`, outside `#if DEBUG`. This is the same logic currently in `debugCopyIOSurfaceCGImage()` (line 9111-9151).
- **Keep** `debugCopyIOSurfaceCGImage()` as a thin wrapper that calls `copyIOSurfaceCGImage()`, preserving backward compatibility for the existing debug `panelSnapshot` callers. Alternatively, just rename calls in the debug code to use the new name.
- The `DebugFrameSample` struct and `debugSampleIOSurface()` stay inside `#if DEBUG` -- they're not needed for screenshots.

**Lines affected:** ~9086-9151 in `GhosttyTerminalView.swift`

### 2. `Sources/TerminalController.swift` -- Add `v2TerminalScreenshot` handler

**What:** New handler method modeled on `v2BrowserScreenshot` (line 8501-8542).

**New method** `v2TerminalScreenshot(params:) -> V2CallResult`:

1. Extract `surface_id` from params (required). Resolve workspace context using the standard `v2` helpers (similar to how `v2BrowserWithPanel` works, but for terminal panels).
2. On `DispatchQueue.main.sync`:
   - Resolve the surface UUID to a `TerminalPanel` via `resolveSurfaceId` + `terminalPanel(for:)`.
   - Call `terminalPanel.hostedView.copyIOSurfaceCGImage()`.
   - If `nil`, call `terminalPanel.surface.forceRefresh()` and retry once (matches existing `panelSnapshot` behavior at line 12650-12652).
   - Convert `CGImage` -> PNG `Data` via `NSBitmapImageRep(cgImage:)` -> `.representation(using: .png)`.
3. Build response dict with `workspace_id`, `surface_id`, `png_base64`, and optionally `width`/`height`.
4. Best-effort write PNG to `$TMPDIR/cmux-terminal-screenshots/` with same pruning logic (reuse `bestEffortPruneTemporaryFiles`). Add `path` and `url` to response if write succeeds.
5. Return `.ok(result)`.

**Note:** Unlike the browser path which goes through `v2BrowserWithPanel` (which resolves browser-specific panels), this needs a terminal-specific resolver. The existing `panelSnapshot` code at line 12640-12641 shows exactly how: `resolveSurfaceId(from:tab:)` -> `tab.terminalPanel(for:)`. This logic should be extracted into a reusable helper or inlined directly in the new method.

### 3. `Sources/TerminalController.swift` -- Register in v2 dispatch + capabilities

**Dispatch** (around line 2225, near the browser commands or in a terminal section):

```swift
case "terminal.screenshot":
    return v2Result(id: id, self.v2TerminalScreenshot(params: params))
```

**Capabilities** (around line 2516, in the `v2Capabilities()` method):

Add `"terminal.screenshot"` to the methods array, placed logically near other terminal/surface commands.

### 4. `CLI/cmux.swift` -- Add `cmux terminal screenshot` subcommand

**What:** New CLI subcommand mirroring `cmux browser screenshot` (line 5138-5278).

This requires understanding how the CLI routes commands. The browser screenshot lives inside the `browser` subcommand handler. A `terminal screenshot` subcommand would:

1. Parse args: `--out <path>` (optional output path), `--json` (JSON output mode).
2. Require a surface ID (via the existing `requireSurface()` pattern or similar).
3. Send `client.sendV2(method: "terminal.screenshot", params: ["surface_id": sid])`.
4. Handle the response with the same copy-or-decode persistence logic as the browser version.
5. Output: `OK <path>` in plain mode, or JSON with `workspace_id`, `surface_id`, `path`, `url`.

**Note:** The CLI file already has helper functions (`bestEffortPruneTemporaryFiles`, `persistPayloadScreenshot`, `syncScreenshotLocationFields`, etc.) inside the browser screenshot block. These are local functions. They should either be extracted to shared scope or duplicated (they're small). Extraction is cleaner.

## Files Changed Summary

| File | Change | Scope |
|------|--------|-------|
| `Sources/GhosttyTerminalView.swift` | Move `copyIOSurfaceCGImage()` out of `#if DEBUG` | ~20 lines moved |
| `Sources/TerminalController.swift` | Add `v2TerminalScreenshot` handler | ~50 new lines |
| `Sources/TerminalController.swift` | Add dispatch case + capability | 2 lines |
| `CLI/cmux.swift` | Add `terminal screenshot` subcommand | ~100 lines (mostly mirrored from browser) |

## What Stays Unchanged

- `debugCopyIOSurfaceCGImage()` and `debugSampleIOSurface()` remain in `#if DEBUG` (the debug method becomes a thin wrapper or is updated to call the new production method).
- `panelSnapshot` / `debug.panel_snapshot` commands remain as-is (DEBUG-only, with pixel diff).
- `browser.screenshot` is untouched.
- `framebufferOnly = false` on the Metal layer is already set -- no rendering changes needed.
- No new permissions required (reads the app's own IOSurface).

## Risks / Edge Cases

- **Empty surface on first capture:** Already handled by the retry-after-`forceRefresh` pattern from the existing `panelSnapshot` code.
- **Surface mid-detach:** If the terminal is closing while a screenshot is requested, `copyIOSurfaceCGImage()` returns `nil` gracefully (layer contents are nil). The handler returns an error.
- **Large terminals:** A 4K terminal surface at 32 bpp is ~33MB raw, but PNG compression brings this down to typically <1MB. The base64 encoding in the response adds ~33% overhead. This matches how `browser.screenshot` already works.
