# Item 15: keyboard, states, scaling, performance

12 October 2026. Windows 11 (build 26100), 2560×1440 at 100% OS scale, release build for
performance numbers. Synthetic fixtures only; no physical reader or personal library.

**How to read the results.** Every row says how it was checked:

- **Test**: a unit or update-level test in `cargo test -p rulibre-ui` (real `update`, fake services).
- **Probe**: a native iced window driven by an example; screenshots under `docs/ui-item-15/`.
- **Not run**: needs a person at the keyboard or hardware. The tools used for this item can
  not send OS-level key presses to a build that is not a registered desktop application, so
  the physical-keyboard rows below are open until someone ticks them.

## Manual run

```powershell
python scripts/make-ui-fixture.py <folder>   # needs Pillow
$env:RULIBRE_UI_DATA_DIR = "<folder>"        # config.toml, ui.toml and covers stay there
cargo run -p rulibre-ui --locked
```

`RULIBRE_UI_DATA_DIR` keeps configuration, preferences and the cover cache out of the user's
profile. The app starts, reads `<folder>/config.toml` (`library_path = '<folder>/Calibre Library'`)
and scans. Startup was executed on Windows with a 240-book fixture (Cyrillic and Polish titles,
JPEG covers on six of seven books, OPF on every fifth); the process ran with the fixture and
logged only the known system-font warning (`mstmc.ttf`). No key was pressed in that run.

## 1. Keyboard

Shortcuts are ignored while a text field has focus (the app only sees events a field did not
capture), except Esc. Letter shortcuts follow the Latin letter when there is one and the physical
key otherwise, so they work on the Russian layout. Held keys repeat movement only.

| Key | Expected | Checked |
|---|---|---|
| `/` | Opens Search; in Search focuses the field. Works on the Russian layout (`.` key). | Test (keymap table, app flow). Not run: physical |
| Up / Down | Moves the cursor one row (list) or one row of cards (grid); Left / Right one card (grid only). | Test |
| PageUp / PageDown | Moves by the visible rows (list) or visible card rows (grid). | Test |
| Home / End | First / last book. | Test |
| Shift + movement | Extends the range from the anchor. | Test |
| Ctrl/⌘ + A | Selects all visible books. | Test |
| Enter | Opens the details pane for the selection (first book if none). In Search: selects the first result, then shows the selection in the library. | Test |
| Esc | Closes, in order: export prompt, reader menu, Convert menu, warning list; then leaves Search (back to the view it came from); then clears the selection. Also works while the Search field has focus. | Test. Not run: physical |
| `T` | Sends the selection. When it cannot, a toast says why (no selection, no reader, several readers, read-only). Not repeated while held. | Test |
| `C` | Opens the Convert menu (opens the details pane if hidden). | Test |
| Ctrl/⌘ + B, Ctrl/⌘ + I | Toggles sidebar, details; saved to `ui.toml`. I is inactive outside Home. | Test |
| Tab / Shift + Tab | Moves between text fields. | Test (key mapping). Not run: physical |
| Tooltips | Sidebar toggle, details toggle, Search, Send, Convert show their shortcut. | Code review only. Not run |
| Setup, open prompt or menu | Shortcuts other than Esc and Tab do nothing. | Test |

Limitations found while doing this:

- iced 0.14 buttons and rows cannot take keyboard focus, so there is no Tab order across buttons.
  The keyboard cursor in the book list and grid has a visible outline (accent border, 1.5 px) that
  appears after a keyboard move and disappears on click. Text fields already show an accent border
  when focused. Probe capture: `docs/ui-item-15/scale-100-grid.png`.
- The Convert menu is an inline section of the details pane, not a popover, so it has no
  click-outside close; Esc and selecting another book close it. The reader menu closes on an
  outside click, as before.

## 2. States

| State | Text and next action | Checked |
|---|---|---|
| Empty library | "No books found" · Rescan, Change library | Probe (item 7 capture) |
| Search, nothing typed | "Type to search by title or author" | Probe (item 8) |
| Search, no results | "No results. Try another title or author." | Probe (item 8) |
| Filter with no books | "No books match this filter. Choose another format or author." | Code review |
| Scan failed | Error text · Retry scan, Change library | Test |
| Scan cancelled | "Scan cancelled. Scan again to load your books." · Retry scan, Change library | Code review |
| No metadata | Details show catalog title and author, then "This book has no further details. Calibre can add them." | Probe (`scale-*-grid.png`, fixture books have no OPF) |
| Metadata unreadable | "Metadata could not be read", cause in a tooltip; logged | Test |
| Missing cover | Colour placeholder with the title (grid, list); "No cover" in details | Probe (fixture, every seventh book) |
| Reader read-only | Warning text in the bar; Send disabled; `T` toast "This reader is read-only, so books cannot be sent." | Test |
| No reader | "No reader connected" (now shown whole at 760 px) · Select folder manually… | Probe (`scale-100-*.png`) |
| Converter missing | In the Convert menu: "No converter found. Install Calibre, then choose Refresh converters." Routes show "no route" with the reason | Code review. Not run: needs a machine without the tools |

No user-facing text uses "successfully"; success toasts read "Sent to …" and "Converted · saved to …".

## 3. Scaling

OS scale cannot be changed without altering the user's desktop, so 150% and 200% are emulated:
`examples/scaling_probe.rs` applies iced's own scale factor and iced enlarges the window by it, so
the layout area stays at the 760×520 minimum, as it would at that OS scale. This is not an OS DPI
run: per-monitor scale changes, DPI-aware window chrome and text hinting by the OS were not exercised.

| Scale | Capture | Layout | Images |
|---|---|---|---|
| 100% | `scale-100-grid.png`, `scale-100-list.png` | No clipping; titles ellipsized by width | Sharp (thumbnail is larger than the card) |
| 150% | `scale-150-*.png` | Same layout, no clipping | Covers softer than at 100% |
| 200% | `scale-200-*.png` | Same layout, no clipping | Covers visibly soft: the 192×264 thumbnail is enlarged to about 320 px |

Finding, not fixed: cover thumbnails are a fixed 192×264 pixels (item 9), so at 150–200% (and at
cover size 200 px above about 100%) they are enlarged by the GPU. Text, icons and the placeholder
stay sharp (vector). Making covers sharp needs a larger or scale-dependent thumbnail, which costs
memory (a 384×528 thumbnail is four times the 300-entry in-memory cache) and disk (the PNG cache
is not bounded). That is a trade-off decision and is left open.

Fixed in this pass: "No reader connected" was cut to "No reader con…" at 760 px.

## 4. Performance (20,000 books)

`cargo run --release -p rulibre-ui --example perf_probe --locked`: a real window, the real
services and a 20,000-folder library on disk; isolated config. Three runs; numbers in milliseconds.

| Measure | Budget | Result (three runs) |
|---|---|---|
| Window start to first frame (cold GPU init included) | none set | 611–660 (first ever run: 1,181) |
| Scan of 20,000 folders | none set | 2,336–2,344 |
| Scan result to next redraw | under 500 | 0.4–1.8 (view built in under 0.2) |
| Search per keystroke, update to results (11 keystrokes, up to 20,000 matches) | under 16 | mean 9.6–10.6, max 11.2 |
| Keyboard cursor move, `update` only (300 presses) | under 16 | mean 0.07–0.08, max 0.4 |
| Shift+End selecting 19,701 books | under 16 | 1.5–1.7 (was 106.9) |
| List scroll, redraw interval (209 frames) | none | mean 5.53–5.55, p95 5.72–5.75, max 6.1 |
| Grid scroll, redraw interval (209 frames) | none | mean 5.55, p95 5.72–5.74, max 5.9 |

Notes on measurement:

- Redraw intervals of 5.55 ms are the display's refresh period (180 Hz): frames are not slower
  than vsync, but this probe cannot show how much headroom is left. "Next redraw" is when iced
  starts the next frame, not when pixels reach the screen, and the keystroke round trip includes
  task scheduling, not input latency. Grid scrolling uses placeholders; covers with a real library
  were measured in item 9.
- Two fixes came out of this pass. Shift-click or Shift+End over thousands of books was
  quadratic in `Selection` (107 ms for 19,701 books, seconds for larger ranges); it is now linear
  (test `huge_range_selection_stays_linear_and_keeps_chronology`, 50,000 books). Keyboard movement
  no longer copies the whole visible order on each press.
- Visible-slice code (`visible_slice_with_height`, `Geometry::book_range`) returns ranges and
  allocates nothing. Per frame, the view builds only the visible rows; their elided strings come
  from a bounded cache. Switching to an author facet still lower-cases every author once per
  switch (not per frame).
- Debug builds are slower; the unit-test budgets in `catalog.rs` apply there.

## 5. Logging

Run with `RUST_LOG` to change the level; the default is `warn`. Logged and not shown: cover
decoding failures (once per book), reader polling failure (once per distinct text) and detection
warnings (once per warning), unreadable metadata, scan warnings (count and the first), window
blur refusal. Shown to the user and not repeated in the log: job results and errors.

## Fixture

`scripts/make-ui-fixture.py` writes 240 books (8 authors, ten Cyrillic and Polish titles, EPUB plus
PDF on every third, `cover.jpg` unless `i % 7 == 0`, `metadata.opf` on every fifth) and a
`config.toml` pointing at them. The library is read-only to the app.
