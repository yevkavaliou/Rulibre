# Item 6: theme, icons and shell

12 October 2026. Windows, 100% scaling. Native iced 0.14 renderer captures.

The production shell uses the mockup's dark tokens and dimensions: toolbar 34 px,
sidebar 148 px, details 226 px, bottom bar 46 px. The centre has 386 px at
760×520 with both side panes visible, and 726 px at 1100×700. Vertical main-area
height is 440 px and 620 px respectively. Hidden panes are omitted from the row.
The window stays opaque, following decision D9; overlay alpha tokens are retained.

`theme.rs` owns colours, spacing/radius tokens, the custom iced theme and container,
button, text-input, scrollable and checkbox styles. The mockup has no error token;
errors currently share its warning colour. `icons.rs` embeds thirteen SVG icons,
with runtime tinting and ISC attribution in `assets/icons/LICENSE`. No icon download
is required at runtime. The mockup itself uses Tabler; the planned native icons use
Lucide-style geometry.

Home, Search, Jobs and Settings navigation allows inspection of the shell. Library,
Formats/Authors, list/grid, book details and Send are static placeholders for their
later items. Setup and the existing scan/cancel/retry and scan Jobs flows are kept.
Sidebar/details toggles take effect immediately, save through Services to `ui.toml`,
coalesce rapid changes and revert to the last saved preferences on failure. The
failure is recorded in notices; the visual toast remains item 11. Details are only
available on Home/Library; Setup also hides the sidebar. Returning from another
screen preserves pane preferences.

## Visual comparison

Inspected the renderer captures below against the dimensions/colours in
`docs/design/mockup.html` and the existing browser reference capture
`docs/spike-results/mockup-settings.png`. The fixed side widths, toolbar, bottom
bar, dark surfaces, sidebar spacing and flexible centre match those shell
proportions. At minimum width, the header, details hint and bottom button fit.
The empty content and details are intentionally placeholders in this item.

Opening the local HTML through the browser tool was blocked by its URL policy
(`file:` is unsupported). No fresh browser screenshot was captured; the comparison
uses the existing reference screenshot and the current CSS. The saved mockup
reference shows Settings, so Home's details width is compared to its CSS.

- [Home, 1100×700](ui-item-6/home-1100.png)
- [Home, 760×520](ui-item-6/home-760.png)
- [Sidebar hidden](ui-item-6/sidebar-hidden-760.png)
- [Both panes hidden](ui-item-6/panes-hidden-760.png)
- [Search: inactive details toggle](ui-item-6/search-760.png)
- [Jobs](ui-item-6/jobs-760.png)
- [Settings: inactive details toggle](ui-item-6/settings-760.png)
- [Setup](ui-item-6/setup-760.png)

## Reproduction and checks

Run `cargo run -p rulibre-ui --example shell_probe --locked`. It creates a real
iced window, captures 1100×700, resizes to 760×520, dispatches toggle/navigation
messages through the production update function, and saves PNGs under
`docs/ui-item-6`. Preferences use temporary isolated paths with RealServices;
the probe asserts their persisted values before closing. It does not load a
library, scan books or interact with a reader. See [probe output](ui-item-6/probe.txt).
These are content-area captures; native window decorations are outside them.

Passed locally:

- `scripts/check-ui.ps1`: workspace fmt check, locked UI build, all-target Clippy
  with warnings denied, all-target tests (32 UI library tests, 1 spike test).
- `cargo clippy --locked --all-targets -- -D warnings`: default workspace.
- `cargo test --locked`: default workspace, all executed tests passed, 3 ignored.
- `git diff --check`.
- Source scan: no literal colour constructors or hexadecimal colour literals in
  production UI source outside `theme.rs`. Spike examples retain their own colours.

New tests cover rapid combined toggles, queued saves, rollback after a failed save,
inactive toggles, restoring Home's details preference, and minimum-width allocation.
Cargo checks ran with access outside the Windows filesystem sandbox, which otherwise
blocks Cargo path canonicalization.

Not verified: mouse input on native controls, tooltips by hovering, higher DPI,
Linux/macOS, personal libraries or physical readers. No remote CI run, commit or PR
was created for this item. Item 7 has not been started.
