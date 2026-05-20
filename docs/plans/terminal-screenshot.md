# Implementation Plan: `terminal.screenshot` Command

> **Status (re-verified):** Plan is still architecturally valid. The core
> primitives the original plan depended on — `debugCopyIOSurfaceCGImage()` on
> `GhosttySurfaceScrollView`, `v2BrowserScreenshot`, `v2BrowserWithPanel`,
> `resolveSurfaceId` + `Workspace.terminalPanel(for:)`, and the CLI's
> `bestEffortPruneTemporaryFiles` / `sanitizedFilenameComponent` helpers — all
> still exist and still work the way the original plan assumed. The only
> updates below are (a) refreshed line numbers, (b) a few small response/API
> shape changes (`workspace_ref`/`surface_ref`, `forceRefresh(reason:)`), and
> (c) the observation that there is still no top-level `terminal` CLI
> subcommand router — it needs to be added.

## Overview

Add a production-ready `terminal.screenshot` v2 socket command and a
`cmux terminal screenshot` CLI subcommand, mirroring the existing
`browser.screenshot` pattern. The core IOSurface capture primitive already
exists (DEBUG-only); this plan promotes it to production and wraps it in the
standard screenshot interface.

## Design Decisions

- **Command name:** `terminal.screenshot` (parallel to `browser.screenshot`).
- **No pixel diff:** Just capture and return the PNG. The debug `panelSnapshot`
  command retains pixel-diff support separately.
- **Debug commands preserved:** Existing `panelSnapshot` /
  `debug.panel_snapshot` commands remain as-is (DEBUG-only, with pixel diff).
- **Response shape parity with browser:** The handler should populate the same
  keys `v2BrowserScreenshot` now returns — `workspace_id`, `workspace_ref`,
  `surface_id`, `surface_ref`, `png_base64`, and (best-effort) `path` + `url`.

## Changes by File

### 1. `Sources/GhosttyTerminalView.swift` — Promote IOSurface capture to production

**What:** Extract `debugCopyIOSurfaceCGImage()` from the `#if DEBUG` block so
it's available in all builds, exposed as `copyIOSurfaceCGImage()`.

- **Current location:** `func debugCopyIOSurfaceCGImage() -> CGImage?` sits at
  approximately lines **12881–12921** inside the `#if DEBUG` block that begins
  near line 12856. The `DebugFrameSample` struct is at ~12857 and
  `debugSampleIOSurface(normalizedCrop:)` is at ~12925.
- **New method:** Add `func copyIOSurfaceCGImage() -> CGImage?` on
  `GhosttySurfaceScrollView`, **outside** `#if DEBUG`. The body is the same
  IOSurface → `CGImage` logic currently in `debugCopyIOSurfaceCGImage()`.
- **Compatibility:** Either (a) keep `debugCopyIOSurfaceCGImage()` as a thin
  `#if DEBUG` wrapper that calls `copyIOSurfaceCGImage()`, or (b) update the
  one debug call site in `panelSnapshot` (see file 2 below, line ~15796) to
  use the new name directly. Option (b) is cleaner — there's exactly one
  caller today.
- `DebugFrameSample` and `debugSampleIOSurface(normalizedCrop:)` stay inside
  `#if DEBUG` — they're not needed for the screenshot path.

### 2. `Sources/TerminalController.swift` — Add `v2TerminalScreenshot` handler

**Reference implementation:** `private func v2BrowserScreenshot(params:)` at
~lines **11117–11158**. Note this is a small handler; everything else in the
file is mostly other browser commands.

**New method** `private func v2TerminalScreenshot(params: [String: Any]) -> V2CallResult`:

1. Resolve workspace and surface UUID from `params` using the standard v2
   helpers (`v2String(params, "surface_id")`, plus the workspace resolution
   that `v2BrowserWithPanel` does — there is no equivalent
   `v2TerminalWithPanel` yet, so either inline the resolution or introduce one).
2. On the main actor (use the existing `v2MainSync { … }` helper — see
   `panelSnapshot` at line ~15782 for the pattern, or the `v2AwaitCallback`
   style used by `v2BrowserScreenshot`):
   - From the resolved tab/workspace, look up the surface via
     `resolveSurfaceId(from: arg, tab: ws)` (line ~16222) and then
     `ws.terminalPanel(for: panelId)` (the same pattern is used at lines
     ~15788–15789 in `panelSnapshot`, and at ~7497, 7563, 7616, 7676, 14075,
     14880, 14932, etc.).
   - Call `terminalPanel.hostedView.copyIOSurfaceCGImage()`.
   - If `nil`, call
     `terminalPanel.surface.forceRefresh(reason: "v2.terminalScreenshot.retry")`
     and retry once. **Note:** `forceRefresh` now takes a `reason:` String
     parameter (see line ~15799); the original plan's `forceRefresh()`
     signature is out of date.
   - Convert `CGImage` → PNG `Data` via
     `NSBitmapImageRep(cgImage:).representation(using: .png, properties: [:])`.
     The existing helper `v2PNGData(from:)` used by `v2BrowserScreenshot` may
     also be appropriate.
3. Build response dict matching `v2BrowserScreenshot`'s current shape:
   ```swift
   var result: [String: Any] = [
       "workspace_id": ws.id.uuidString,
       "workspace_ref": v2Ref(kind: .workspace, uuid: ws.id),
       "surface_id":   surfaceId.uuidString,
       "surface_ref":  v2Ref(kind: .surface, uuid: surfaceId),
       "png_base64":   imageData.base64EncodedString(),
   ]
   ```
   (The original plan omitted `workspace_ref` / `surface_ref`; they were added
   to the browser variant since the plan was written and the terminal variant
   should match.)
4. Best-effort write the PNG into
   `$TMPDIR/cmux-terminal-screenshots/surface-<shortSid>-<ts>-<rand>.png`
   using the same idiom as `v2BrowserScreenshot` at lines ~11140–11154 —
   including the call to `bestEffortPruneTemporaryFiles(in:)`. If the write
   succeeds, set `result["path"]` and `result["url"]`.
5. Return `.ok(result)`.

### 3. `Sources/TerminalController.swift` — Register in v2 dispatch + capabilities

**Dispatch** (line ~3060, immediately after the `case "browser.screenshot":`
arm):

```swift
case "terminal.screenshot":
    return v2Result(id: id, self.v2TerminalScreenshot(params: params))
```

**Capabilities** (the `v2Capabilities()` method begins at line ~3267; the
existing methods list contains `"browser.screenshot"` at ~3392 and
`"debug.panel_snapshot"` / `"debug.panel_snapshot.reset"` at ~3485–3486).
Add:

```swift
"terminal.screenshot",
```

near the other terminal/surface entries.

### 4. `CLI/cmux.swift` — Add `cmux terminal screenshot` subcommand

**What:** New CLI subcommand mirroring `cmux browser screenshot`. The browser
screenshot block lives inside `runBrowserCommand` at approximately lines
**8573–8712** (the original plan referenced 5138–5278 — the file has grown
substantially since then).

**Two pieces of work in this file:**

1. **Add a `terminal` top-level dispatch case.** The top-level dispatch is in
   the big `switch command` block around lines 3884 ff. (`case "browser":`
   calls `runBrowserCommand`). There is **no** `case "terminal":` today, so a
   new dispatch arm and a new `runTerminalCommand(commandArgs:client:jsonOutput:idFormat:)`
   method must be introduced. Model the entry point on `runBrowserCommand`
   (declared at line ~7747).

2. **Implement the `screenshot` subverb inside `runTerminalCommand`.** This is
   essentially a direct port of lines 8573–8712:
   - Parse args: optional `--out <path>`, optional `--json` flag.
   - Resolve the surface ID via the existing `requireSurface()` closure /
     helper pattern used throughout `runBrowserCommand` (line ~7802 declares
     it inside `runBrowserCommand` — the same pattern can be reproduced in
     `runTerminalCommand`).
   - `client.sendV2(method: "terminal.screenshot", params: ["surface_id": sid])`.
   - Reuse the response-handling pipeline: `writeScreenshot`,
     `syncScreenshotLocationFields`, `persistPayloadScreenshot`. Today these
     are **nested** inside the browser screenshot block as local functions.
     Two reasonable options:
       1. Duplicate them inside the terminal block (they're small and the
          duplication is contained).
       2. Lift them to file-scope helpers and call from both. Cleaner long
          term.
   - `bestEffortPruneTemporaryFiles` (line ~3952) and
     `sanitizedFilenameComponent` (line ~3942) are already class-scoped
     `private` methods and can be reused directly — no extraction work
     required for those.
   - Default temp dir should be `cmux-terminal-screenshots-cli` (parallels the
     browser variant's `cmux-browser-screenshots-cli` at line ~8674).
   - Output: JSON payload when `--json`/`effectiveJSONOutput`, else
     `OK <url-or-path>`.

3. **Help text:** Add `terminal screenshot [--out <path>] [--json]` to the
   `usage()` string (the browser variant is in the usage block around line
   ~26258).

## Files Changed Summary

| File | Change | Approx scope |
|------|--------|--------------|
| `Sources/GhosttyTerminalView.swift` | Move IOSurface capture out of `#if DEBUG`, rename to `copyIOSurfaceCGImage()` (~lines 12881–12921). | ~20 lines moved/renamed |
| `Sources/TerminalController.swift` | Add `v2TerminalScreenshot` handler near `v2BrowserScreenshot` (~line 11158). | ~50 new lines |
| `Sources/TerminalController.swift` | Add `"terminal.screenshot"` dispatch case (~line 3061) and capability entry (in `v2Capabilities`, ~line 3267, alongside the entries near 3392/3485). | ~2 lines |
| `CLI/cmux.swift` | Add top-level `case "terminal":` (~line 3884), implement `runTerminalCommand` modeled on `runBrowserCommand`, port screenshot handling (~lines 8573–8712), update `usage()` (~line 26258). | ~100–150 lines depending on helper extraction |

## What Stays Unchanged

- `debugCopyIOSurfaceCGImage()` and `debugSampleIOSurface(normalizedCrop:)`
  remain in `#if DEBUG` (or `debugCopyIOSurfaceCGImage()` becomes a thin
  wrapper around the new production method).
- `panelSnapshot` / `debug.panel_snapshot` /
  `debug.panel_snapshot.reset` commands remain as-is (DEBUG-only, with pixel
  diff via `panelSnapshots` ring at line ~15667).
- `browser.screenshot` and its CLI counterpart are untouched.
- `framebufferOnly = false` on the Metal layer is already set — no rendering
  changes needed.
- No new permissions required (reads the app's own IOSurface).

## Risks / Edge Cases

- **Empty surface on first capture:** Already handled by the retry-after-
  `forceRefresh(reason:)` pattern from the existing `panelSnapshot` code
  (lines 15796–15801).
- **Surface mid-detach:** If the terminal is closing while a screenshot is
  requested, `copyIOSurfaceCGImage()` returns `nil` gracefully (layer
  contents are nil). The handler returns an `internal_error` /
  `not_found` response (match `v2BrowserScreenshot`'s error codes for
  consistency).
- **Large terminals:** A 4K terminal surface at 32 bpp is ~33MB raw; PNG
  compression brings this down to typically <1MB. The base64 encoding in the
  response adds ~33% overhead. This matches how `browser.screenshot` already
  behaves.
- **Helper duplication vs extraction in the CLI:** Be deliberate. The browser
  screenshot block has accumulated ~140 lines of nested helpers; copy-pasting
  them into a terminal sibling will create a second copy that future fixes
  must update in two places. Extracting `writeScreenshot`,
  `persistPayloadScreenshot`, `syncScreenshotLocationFields`, `fileURL`, and
  `hasText` into file/class scope is the cleaner refactor and is recommended
  before the terminal subcommand is added.

## Verification After Implementation

- Build with `./scripts/reload.sh --tag terminal-screenshot`.
- From a tagged Debug app, exercise via
  `CMUX_TAG=terminal-screenshot scripts/cmux-debug-cli.sh terminal screenshot --json`
  and confirm the resulting `path` opens to a non-blank PNG.
- Confirm `cmux terminal screenshot --out /tmp/x.png` writes a file and
  prints `OK /tmp/x.png`.
- Confirm `debug.panel_snapshot` still works (regression check that the
  rename/wrapper didn't break the existing pixel-diff path).
