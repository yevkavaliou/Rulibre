# Window opacity and blur (item №14)

Settings has two rows modelled on Sonora:

- **Window opacity**, 10–100 % in steps of 5, default 70 %. 100 % is an opaque
  window. Lower values let the desktop show through the whole window (root fill and
  details panel alpha = the value).
- **Blur window**, a toggle, default on. With an opacity below 100 % it draws the
  window over a *blurred* desktop (Windows acrylic). Off: a translucent window over a
  sharp desktop. Needs an opacity below 100 %.

Both are saved in `ui.toml` (`window_opacity`, `window_blur`). This supersedes the
"no window blur in v1" verdict of [UI_SPIKE.md](UI_SPIKE.md) and the cover-colour wash
first tried for this item; neither is used any more.

## Why the spike said no

wgpu's default Windows swap chain (`DxgiFromHwnd`) reports only the `Opaque` alpha
mode, so iced content can never be composited with transparency. wgpu 27 can also
present through a DirectComposition visual (`DxgiFromVisual`), which supports
premultiplied alpha. iced 0.14 does not expose that choice. Two small, marked
patches in [vendor/](../vendor/README.md) make it reachable:

- `iced_wgpu`: build the instance from `WGPU_*` environment options and prefer
  premultiplied alpha when `WGPU_DX12_PRESENTATION_SYSTEM` is set.
- `iced_winit`: create the transparent window without a redirection bitmap
  (`WS_EX_NOREDIRECTIONBITMAP`); otherwise an opaque white surface sits under the
  swap chain. This explained the flat grey window seen in the first attempt
  (dark 35 % fill over white = 174).

## How it works

- `blur::prepare_environment()` runs first in `main` on Windows and sets
  `WGPU_BACKEND=dx12` and `WGPU_DX12_PRESENTATION_SYSTEM=visual` unless the user
  already chose values. The window is created `transparent` only when that
  configuration is in force (`blur::ready()`).
- `app::update` ends with `sync_blur`: when blur is on and the opacity is below
  100 %, a task applies the acrylic backdrop, otherwise it clears it. The result is
  `Message::BlurApplied`.
- **Acrylic is applied with `SetWindowCompositionAttribute`** (accent policy 4), not
  with the DWM system backdrop that `window-vibrancy::apply_acrylic` selects on
  Windows 11 22H2+. The system backdrop turns into a solid fallback when the window
  loses focus; the accent policy keeps blurring. Measured with the real window
  focused and unfocused over a striped backdrop: the system backdrop changed a lot
  (mean chroma 26 → 70), the accent policy hardly at all (11.6 → 8.8). Its tint is
  light (18, 18, 21, 60); the fill alpha does the darkening. See
  [opacity-15-unfocused](ui-item-14/opacity-15-unfocused.png).
- Fills stay opaque until the effect is really in place (no transparent flash): with
  blur on, until the OS confirms the backdrop; without blur, a translucent window
  needs no backdrop and is applied at once. The frame is cleared transparent
  (`app::style`) only while translucent. While translucent the fills use a darker
  "glass" palette (14, 14, 16 / 20, 21, 24): the acrylic layer lifts any backdrop
  (about 52/255 over black), so the normal palette looked grey and milky.
- Windows with this mode use their own title bar: the toolbar row is the title bar
  (drag area, double-click to maximise, minimise/maximise/close buttons) and the OS
  caption is removed (`decorations: false`, undecorated shadow). An undecorated
  window has no resize border, so thin invisible handles around the window call
  `window::drag_resize`. The native caption otherwise stays; as a separate strip it
  was visibly lighter than the translucent content. This revises decision D3 for
  this configuration only. Snap layouts on the maximise button are lost.
- If the OS refuses (for example an old Windows 10 build) the window stays opaque
  for the session and one toast says so. On other platforms and when the environment
  is not ready, both controls are disabled with an explanation; the values are still
  stored.
- `SetWindowCompositionAttribute` is an undocumented but long-stable user32 export,
  called through `libloading` in a small `unsafe` block in `blur.rs`.

## Tests and checks

- `blur::tests::alpha_is_opaque_until_active_and_follows_opacity`.
- `app::tests::window_translucency_follows_opacity_and_blur_and_never_flashes`:
  nothing requested when not ready, opaque until confirmed, live slider with a 10 %
  floor and no OS calls, blur off is translucent at once and clears the backdrop,
  100 % is opaque, a refusal stays opaque with one toast and still saves.
- `settings` tests cover defaults, clamping and persistence of both values.
- Visual check: `scripts/capture-blur.ps1 -Output file.png -Opacity 15 [-Inactive]`
  shows the real application view in a transparent window over a striped backdrop and
  captures the desktop at OS level (iced screenshots cannot include the backdrop).
  `-Inactive` hands the focus to the backdrop window first. Captures at 15, 40, 70 and
  100 %: [15](ui-item-14/opacity-15.png), [40](ui-item-14/opacity-40.png),
  [70](ui-item-14/opacity-70.png), [100](ui-item-14/opacity-100.png), and unfocused
  at 15 %.

Verified: Windows 11 build 26100, NVIDIA RTX 5060 Ti, debug build. Not verified:
Windows 10, physical mouse use of the custom title bar (drag, double-click, buttons,
edge resizing: wired to iced window operations and compiled, not exercised with
real input), flicker while resizing, battery/power behaviour, other GPUs and drivers,
150 %/200 % scaling, HDR, macOS (vibrancy not wired) and Linux (opaque).
