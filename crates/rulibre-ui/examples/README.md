# Item №2 capability lab

`spike_capabilities.rs` is a disposable experiment, excluded from the shipped
application. Results and limitations: [UI_SPIKE.md](../../../docs/UI_SPIKE.md).

Interactive: `cargo run -p rulibre-ui --example spike_capabilities --locked`.
Try the effect buttons, Ctrl/Shift clicking the test button, resize and scroll.

Automated Windows measurement: `./scripts/measure-ui-spike.ps1` (optionally
`-Backend dx12` or `-Backend software`). Each run decodes 200 distinct synthetic
240×360 JPEGs, scrolls the grid, switches effects on the same window, captures
renderer screenshots and exits. Measurements go to `target/ui-spike` by default.
Screenshots capture iced content, **not** desktop composition or OS blur.

The synthetic fixture encoding precedes the timed decode. Handles retain RGBA
pixels; the image renderer may upload textures lazily when they become visible.
Redraw timestamps measure event intervals, not GPU execution or presented frames.
The lab deliberately forces redraws; this is not the production idle loop.

`assets/book-open.svg` comes from Lucide (downloaded 12 October 2026). Its original
license and Feather notices are preserved in `assets/LUCIDE-LICENSE`.
The image decoder is also used by the production cover pipeline. Window
vibrancy remains a dev dependency. `scripts/check-ui.ps1` includes
the example's tests; it does not open windows in CI.

## Item №5 setup and scan probe

`cargo run -p rulibre-ui --example setup_scan_probe --locked` creates a temporary
20,000-book folder fixture and opens the real iced application view. It verifies
first-run Setup, validates and saves the chosen path, scans through RealServices
and the coordinator, then loads the saved config and scans again. It measures
16 ms heartbeat delivery while the scan runs, saves renderer screenshots in
`target/ui-item-5`, closes the window and removes the temporary fixture.

The program chooses the fixture through messages; it does not automate Browse
or physical mouse/keyboard input. A fixture `metadata.db` is only the marker
required by setup validation, not a SQLite database. No personal library,
configuration or reader is accessed. Results: [UI_SETUP.md](../../../docs/UI_SETUP.md).

## Item №7 catalog probe

`cargo run -p rulibre-ui --example catalog_probe --locked` renders and scrolls a
synthetic 20,000-book catalog through the production Home view. It records native
redraw intervals, then captures list/selection/warnings/empty/grid placeholder
states at 1100×700 and 760×520 under `docs/ui-item-7`. Preferences are isolated in
a temporary directory; no personal library or reader is accessed. See
[UI_CATALOG.md](../../../docs/UI_CATALOG.md) for measurements and limitations.

## Item №9 cover probe

`cargo run -p rulibre-ui --example covers_probe --locked -- <library> <cache>`
scans the supplied library read-only and repeats its covered books into 5,000
distinct catalog IDs. The source JPEGs are real; the catalog is a load fixture,
not 5,000 distinct books. Use an isolated cache outside the library (for example
`target/ui-item-9-covers-cache`). Run twice with the same cache to measure reuse.
The native probe traverses the grid, checks the 300-entry/four-worker limits,
records 16 ms heartbeat delivery and saves grid/list screenshots under
`docs/ui-item-9`. Preferences are temporary; no physical input is simulated.
See [UI_COVERS.md](../../../docs/UI_COVERS.md) for results and limitations.

## Item №12 Settings probe

`cargo run -p rulibre-ui --example settings_probe --locked` captures Settings
at 760×520 with default preferences, saved toggles and a long export path, and a
scripted validation toast. Configuration and library paths are temporary.
Images are saved under `docs/ui-item-12`; the window closes automatically.
It uses the real settings writer but does not drive the native folder picker.
See [UI_SETTINGS.md](../../../docs/UI_SETTINGS.md) for verification and limitations.

## Item №14 window blur probe

`scripts/capture-blur.ps1 -Output file.png -Opacity 30` (Windows only) builds and runs
`blur_probe`, the real application view in a transparent window with the acrylic
backdrop, over a striped full-screen window, and saves an OS-level capture. iced's
own screenshots cannot include the backdrop. Results: [UI_BLUR.md](../../../docs/UI_BLUR.md).

## Item №15 performance and scaling probes

`cargo run --release -p rulibre-ui --example perf_probe --locked` builds a 20,000-folder
library in a temporary directory, scans it through the real services and prints first
frame, scan, scan-result-to-redraw, search round trip per keystroke, keyboard cursor cost,
range selection and list/grid scroll redraw intervals. Configuration is temporary.

`cargo run -p rulibre-ui --example scaling_probe --locked` captures the shell at iced scale
factors 1.0, 1.5 and 2.0 (emulation of 100/150/200% OS scale; the layout area stays 760×520)
with generated JPEG covers, into `docs/ui-item-15`. Results and limits:
[UI_CHECKLIST.md](../../../docs/UI_CHECKLIST.md).
