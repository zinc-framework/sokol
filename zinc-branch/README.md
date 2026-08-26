# Zinc branch — deviations from upstream sokol

This is a fork of [floooh/sokol](https://github.com/floooh/sokol) maintained for the
[Zinc](https://github.com/zinc-framework) framework. The `zinc` branch is the one Zinc
consumes (see `Zinc.Bootstrapper/.gitmodules`).

**The fork's default position is zero divergence.** Everything here is a deliberate
exception, and each one is expected to end either upstream or deleted. If you're looking
at this folder to decide whether to add something, prefer:

1. an upstream PR,
2. then a Zinc-side solution outside sokol (see *Rejected deviations* below),
3. and only then a patch here.

## Current deviations

| Feature | Files touched | Status |
|---|---|---|
| [win32 `composite_mode`](win32-composite-mode.md) | `sokol_app.h` | Candidate for upstreaming |

## How to re-sync with upstream

`origin` is upstream, `zinc` is the fork (matching how `sokol_gp` is wired):

```sh
git fetch origin                          # floooh/sokol
git checkout zinc
git rebase origin/master                  # keep deviations as replayable commits
git push --force-with-lease zinc zinc
```

Rebase rather than merge, so each deviation stays a single reviewable commit that can be
dropped the moment upstream absorbs it. After any re-sync, rebuild via
`Zinc.Bootstrapper/build.ps1` and re-check the table above — if a row's feature landed
upstream, delete the patch, delete its page here, and drop the row.

## Rejected deviations

Things that were considered as sokol patches and deliberately solved elsewhere. Recorded
so they don't get re-litigated.

### macOS `CAMetalLayer.displaySyncEnabled` (removed Aug 2026)

The fork previously hard-coded `_sapp.macos.mtl.layer.displaySyncEnabled = NO` to work
around macOS Tahoe input lag ([floooh/sokol#1344](https://github.com/floooh/sokol/issues/1344)).
Upstream's Jul-2026 swapchain update added the same knob as a supported option,
`sapp_desc.metal.disable_display_sync`, so the patch was dropped and Zinc sets the field
instead. **This is the ideal outcome for everything in this folder.**

### Borderless / undecorated windows

sokol_app has no concept of an undecorated window, and Zinc needs one for "desktop
companion" apps. Rather than patch sokol, Zinc restyles the native window handle that
sokol already hands out (`sapp_win32_get_hwnd` / `sapp_macos_get_window`) from its own
native lib — see `Zinc.Bootstrapper/libs/zinc_platform/build/zinc_window.{c,m}`.

That also covers always-on-top, drag-by-content, click-through and taskbar visibility,
which are app policy rather than anything sokol should own.

### Multi-viewport Dear ImGui

`sokol_imgui.h` never sets `ImGuiBackendFlags_PlatformHasViewports` /
`RendererHasViewports`, never fills the `ImGuiPlatformIO` window hooks, and renders only
the main viewport's draw data. sokol_app is single-window by design, so this isn't a patch
— it's a whole platform+renderer backend. Not attempted. Docking itself needs no backend
support and works as-is.
