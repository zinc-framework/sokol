# win32 `composite_mode` (transparent windows on Windows/D3D11)

**Files touched:** `sokol_app.h`
**Upstream status:** not submitted yet — candidate for a PR to floooh/sokol
**Added:** Aug 2026

## What upstream has

The 02-Jul-2026 "advanced swapchain configuration" update added `sapp_desc.composite_mode`:

```c
typedef enum sapp_composite_mode {
    _SAPP_COMPOSITEMODE_DEFAULT,        // default is OPAQUE
    SAPP_COMPOSITEMODE_OPAQUE,          // no blending with background
    SAPP_COMPOSITEMODE_PREMULTIPLIED,   // premultiplied alpha-blending with background
} sapp_composite_mode;
```

Upstream implements `PREMULTIPLIED` for **macOS+Metal** and **WebGPU/emscripten** only. On
the D3D11 backend the field is accepted and silently ignored — the window stays opaque.

## What this branch adds

The same behaviour for the Windows D3D11 backend, so Zinc can build "desktop companion"
apps (transparent, frameless windows that draw over the desktop) on Windows as well as macOS.

## Why it isn't a small patch

A swapchain bound to an HWND **always presents opaque on Windows**, no matter what alpha
the backbuffer contains. There is no flag to change that. Per-pixel window alpha requires
routing the swapchain through DirectComposition instead:

| | upstream (opaque) | this branch (composited) |
|---|---|---|
| window ex-style | `WS_EX_APPWINDOW \| WS_EX_WINDOWEDGE` | `+ WS_EX_NOREDIRECTIONBITMAP` |
| device + swapchain | `D3D11CreateDeviceAndSwapChain` | `D3D11CreateDevice`, then `IDXGIFactory2::CreateSwapChainForComposition` |
| swapchain desc | `DXGI_SWAP_CHAIN_DESC` | `DXGI_SWAP_CHAIN_DESC1`, `AlphaMode = DXGI_ALPHA_MODE_PREMULTIPLIED` |
| presented to | the HWND | an `IDCompositionVisual` under an `IDCompositionTarget` |

`WS_EX_NOREDIRECTIONBITMAP` is the non-obvious requirement: without it the DWM keeps
compositing the window's opaque redirection surface and the window stays solid regardless
of the swapchain's alpha mode.

## Safety properties

- **The opaque path is byte-for-byte unchanged.** It still runs
  `D3D11CreateDeviceAndSwapChain` exactly as before, so the default configuration cannot
  regress.
- **Failure falls back rather than aborting.** If any part of the composition setup fails,
  the code logs `WIN32_COMPOSITEMODE_FELL_BACK_TO_OPAQUE`, releases what it built, and
  continues down the normal HWND path. That path is opaque, so for a transparent app the
  fallback is not graceful: the transparent clear shows as solid black.
- **The debug flag is retried, like the HWND path does.** In `SOKOL_DEBUG` builds the
  device is requested with `D3D11_CREATE_DEVICE_DEBUG`, which fails outright on any machine
  without the Windows "Graphics Tools" optional feature, so on most end-user PCs. The
  composite path first shipped without the retry and fell back to opaque on exactly those
  machines (Oct 2026: a released game's window showed black on players' PCs, while dev
  boxes, which have Graphics Tools, looked fine). It now logs
  `WIN32_D3D11_CREATE_DEVICE_WITH_DEBUG_FAILED` and retries without the flag. To test the
  no-Graphics-Tools case on a dev box, make the debug-flag `D3D11CreateDevice` call fail.
- **No new link-time dependency.** `dcomp.dll` is resolved with
  `LoadLibrary`/`GetProcAddress`, so binaries built with composite support still run on
  systems where DirectComposition is unavailable.

## Implementation notes

**mingw's `<dcomp.h>` does not compile in C mode.** Its C-mode struct declarations are
broken (`must use 'struct' tag`, duplicate members), and sokol.c is C. Only three
interfaces and five methods are needed, so they're declared by hand and called through an
explicit vtable pointer in both C and C++ — the declarations then don't have to match the
compiler's COM ABI, only `dcomp.dll`'s actual layout.

**`IDCompositionVisual::SetContent` is at a fixed vtable slot despite the overloads.**
Several `IDCompositionVisual` methods are overloaded, and MSVC and GCC order overload
groups differently in the vtable — which is exactly why mingw's header carries two
orderings. `SetContent` is not itself overloaded, and each overload pair occupies the same
two slots under either ordering, so its index (15) is stable. The preceding slots are
padded out explicitly with a comment listing what they are.

**Flip-model constraints.** Composition swapchains are always flip-model, which forbids
SRGB *surface* formats and MSAA on the swapchain itself. sokol already separates the
surface format (`_sapp_d3d11_swapchain_color_format`) from the render-target-view format
(`_sapp_d3d11_view_color_format`), so `sapp_desc.srgb` keeps working unchanged.

**Resize.** `_sapp_d3d11_resize_default_render_target` previously read `BufferCount` from
`swap_chain_desc`, which the composition path never fills in. It now uses a recorded
`swap_chain_buffer_count` set by whichever path created the swapchain.

## New log items

```
WIN32_D3D11_CREATE_DEVICE_WITH_DEBUG_FAILED
WIN32_D3D11_CREATE_DEVICE_FAILED
WIN32_D3D11_GET_IDXGIFACTORY2_FAILED
WIN32_D3D11_CREATE_COMPOSITION_SWAPCHAIN_FAILED
WIN32_DCOMP_LOAD_DLL_FAILED
WIN32_DCOMP_CREATE_DEVICE_FAILED
WIN32_DCOMP_CREATE_TARGET_FAILED
WIN32_DCOMP_CREATE_VISUAL_FAILED
WIN32_COMPOSITEMODE_FELL_BACK_TO_OPAQUE
```

## Using it

```c
sapp_desc {
    .composite_mode = SAPP_COMPOSITEMODE_PREMULTIPLIED,
    ...
}
```

Then clear the swapchain to premultiplied transparent black (`{0,0,0,0}`) and draw only
what should be visible. Note *premultiplied*: RGB must already be multiplied by alpha, so
a half-transparent red is `{0.5, 0, 0, 0.5}`, not `{1, 0, 0, 0.5}`.

From Zinc this is `RunOptions.transparentWindow`, which also forces the pass to clear
transparent — see `Zinc/src/Core.cs`.

## Testing

Verified on Windows 11 + D3D11: a transparent window composites correctly over the desktop,
and the opaque default is unchanged.

Note that **GDI screen capture cannot see this**. `BitBlt`/`CopyFromScreen` over a
DirectComposition window returns the window's own surface, not the composited desktop, so
automated screenshot comparison will show black where the desktop should show through.
Verify transparency by eye.
