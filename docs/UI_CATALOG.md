# Item 7: catalog, selection and virtual Home list

12 October 2026. Windows, native iced renderer, debug build, 100% scaling.

Home now displays a sorted catalog in 36 px rows with a thumbnail placeholder,
title, author and format tags. Unicode lowercase keys sort author then title;
BookId breaks ties deterministically and indexes the snapshot. This is Unicode
code-point ordering after lowercase, not locale-specific Polish collation.

Selection uses stable BookIds, an anchor and a cursor. Plain click selects one;
Ctrl/Cmd toggles and changes the anchor; Shift (including Ctrl+Shift) replaces
selection with the range in the supplied visible order. A hidden/missing anchor
falls back to one book. Up/Down moves the cursor, Shift extends, Ctrl/Cmd+A selects
all, Escape clears. Focus loss resets modifiers. Keyboard actions use ignored
iced events, so captured text-input events do not trigger Home shortcuts.
Selection opens the details placeholder. Rescans preserve existing IDs and drop
missing IDs on completion; changing libraries clears IDs scoped to the old one.

The Scrollable renders only the visible slice plus three rows above/below and
two spacers. Responsive width drives the title/author columns; text uses actual
iced paragraph metrics and a binary search for ellipsis. A cache holds at most
2048 text entries, with width/font size in the key. Large format sets show a
count and expose the complete labels in a tooltip. Covers remain item 9.

The Home header includes count, refresh, warning list opener and list/grid
buttons. Empty library has Rescan and Change library. The grid button opens an
explicit placeholder with Show list; its persistence/default grid mode remains
item 9. List is usable by default during this stage. Full metadata/Convert remain
item 13. The new mockup states were approved by the user in this chat before
production implementation.

## Native renderer verification

Run `cargo run -p rulibre-ui --example catalog_probe --locked` from the repository
root. The probe constructs a synthetic 20,000-book catalog, uses the production
view/update and real Scrollable operations, scrolls continuously for 240 redraws,
then captures list, selection, warnings, empty library and grid placeholder. It
uses isolated temporary preferences and does not access a personal library or
reader. Selection messages are driven programmatically, not by physical input.

Final measurement ([raw output](ui-item-7/probe.txt)):

| Measurement | Result |
|---|---:|
| Catalog build, 20,000 books | 56.875 ms |
| Scroll redraw intervals, after 30-frame warmup | 209 samples |
| Mean interval | 5.544 ms |
| p95 interval | 10.429 ms |
| Maximum interval | 11.675 ms |

These are native window redraw event intervals, not GPU execution/presentation
times. They show stable delivery during the automated continuous scroll; they do
not replace a human mouse-wheel check. The original uncached shaping took mean
26.055 ms / p95 31.776 ms, motivating the bounded text cache.

Inspected the native captures at both sizes: columns and format tags fit;
Cyrillic/Polish text renders, long titles ellipsize at minimum width, two selected
rows and the details count agree; warning/empty actions fit above the fixed bar.

- [List at 1100×700](ui-item-7/list-1100.png)
- [Selected rows at 760×520](ui-item-7/list-760-selected.png)
- [Warning list](ui-item-7/warnings-760.png)
- [Empty library](ui-item-7/empty-760.png)
- [Grid placeholder](ui-item-7/grid-placeholder-760.png)

## Checks and limitations

Passed: `scripts/check-ui.ps1` (fmt, locked build, all-target UI Clippy and tests:
40 library tests plus one spike test), default-workspace locked all-target
Clippy and locked tests (107 passed, 3 ignored), `git diff --check`.
Tests cover Unicode sorting/indexing, 20,000-book catalog budget, viewport
boundaries, modifier combinations/range/filtering/removal semantics, keyboard
movement, focus-loss reset, details opening, rescan identity retention and
width-aware Unicode truncation. Cargo/fixture checks ran outside the Windows
filesystem sandbox because it blocks path canonicalization and atomic writes.

No physical mouse/keyboard interaction, higher DPI, Linux/macOS, remote CI or
physical reader was verified here. Native screenshots and redraw measurements
use synthetic catalog data. No commit or PR was created. Item 8 was not started.
