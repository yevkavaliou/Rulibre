# Rulibre iced UI: implementation plan

Date: 12 October 2026.

Repository: https://github.com/yevkavaliou/Rulibre. Branch base: `codex/pre-ui-baseline`.
Prerequisite: [PRE_UI_PLAN.md](PRE_UI_PLAN.md) (all 15 items done; core readiness in [docs/READINESS.md](docs/READINESS.md)).
Design reference: [docs/design/mockup.html](docs/design/mockup.html). It is an interactive HTML mockup; open it in a browser. The layout and behaviour in it are the specification.

This document plans the new desktop UI built with **iced**. The core crate (`rulibre-core`) is already UI-independent; this plan only adds a new client crate, plus the few core additions the design needs (cover discovery).

## Rules for execution

- The order is strict: №1 → №2 → … → №17. Start the next item only after the previous one is complete, including its checks. This applies even where items are technically independent.
- Status of each item is only "done" or "not done". Partial work goes into the notes; the status is not changed.
- Tests are written together with the code, not afterwards. Every item must keep `cargo fmt --check`, `cargo clippy -D warnings` and `cargo test --locked` green for the whole default workspace plus the new crate.
- If an item needs a screen or state that is not in the mockup (see "Design gaps" below), update `docs/design/mockup.html` first and get sign-off before implementing it.
- After finishing an item fill in its post-implementation notes: what changed, which checks were run and their result, commit/PR link if any, and remaining limitations. Never present a check on temporary directories or fakes as a check on a physical reader. Never mark something verified that was not run.
- Do not copy code from Sonora (GPL-3.0). It was used only as a source of layout ideas. Rulibre is MIT.
- The Calibre library stays read-only ([docs/LIBRARY_READ_ONLY.md](docs/LIBRARY_READ_ONLY.md)). No UI action may write into the library folder or `metadata.db`.

## Target design (summary of what was agreed)

```
┌────────────────────────────────────────────────────────────────────────┐
│ [▤]  Rulibre                                                      [▥]  │  toolbar: sidebar toggle (left), details-pane toggle (right)
├───────────┬───────────────────────────────────────────┬────────────────┤
│ Home      │  Home                       7 books  [≡][▦]│  Cover         │
│ Search    │                                           │  Title         │
│ Library ▾ │   list rows  or  cover grid               │  Author, ...   │
│  Formats ▾│                                           │  Description   │
│  Authors ▾│                                           │  [Convert ▾]   │
│ Jobs  (2) │                                           │                │
│ Settings  │                                           │                │
├───────────┴───────────────────────────────────────────┴────────────────┤
│ ● Kobo Libra 2 · E:\ · 12.4 GB free        42% ▬▬▬▬  [Cancel] [Send to reader] │  always visible
└────────────────────────────────────────────────────────────────────────┘
```

Agreed behaviour:

- **Three panes plus a fixed bottom bar.** Left sidebar (hideable, fully removed from layout when hidden), centre content, right details pane (hideable via the toggle at the top right). The bottom bar can never be hidden.
- **Sidebar items:** Home, Search, Library (dropdown containing Formats and Authors dropdowns), Jobs (badge with the number of unfinished jobs), Settings. Home is the default screen and shows all books.
- **Home / Library filter screens:** list or cover grid (user toggle, grid is default in the mockup). Selecting a Format or Author item shows the same layout filtered, titled "Library · EPUB".
- **Search** is a separate screen: search field at the top, results as a list below, details pane hidden. Nothing is shown until the user types. Filtering is live.
- **Details pane:** shows the selected book. Selecting a book always opens the pane, even if the user had hidden it. With several books selected it shows "N selected", a short title list and Convert for all of them. Multi-select is keyboard-only for now (Ctrl+click toggles, Shift+click selects a range).
- **Convert** lives in the details pane, not in the bottom bar. Routes that have no tool are shown as "no route".
- **Bottom bar** shows only the connected reader: none / one / several (then the user must choose). It shows free space, transfer progress with Cancel, and "Send to reader" (or "Send N to reader" for a multi-selection, queued one after another).
- **Jobs** is a screen: Running, Up next, Done, per-job Cancel, "Clear finished".
- **Settings** is a screen: window blur, cover tint, prefer KEPUB for Kobo, library folder.
- **Visual effects (configurable):** OS window blur (Mica/Acrylic/vibrancy) and a blurred cover tint behind the details pane. Per-element backdrop blur (Sonora's GPUI feature) is explicitly **not** planned: iced has no equivalent; flat translucent fills are used instead.
- Dark theme only in the first version.

## Decisions taken (recorded so they are not reopened by accident)

| № | Decision | Why |
|---|---|---|
| D1 | Use **iced 0.14** (crates.io), not GPUI. | Stable released crate, documented, already named in the original plan. GPUI is an unreleased git dependency with a fork to maintain. |
| D2 | New crate `crates/rulibre-ui`. The Tauri/Leptos `rulibre-gui` stays until the new UI reaches parity and is removed in №17. The TUI (`rulibre`) stays. | Keeps the repository buildable and gives a fallback during development. |
| D3 | Native window decorations; the toolbar in the mockup is an in-app row below the OS title bar. | A custom title bar needs drag, resize, snap and DPI handling per OS. Not worth it for v1. |
| D4 | UI preferences live in a separate `ui.toml`, not in core `Config`. | The TUI shares core `Config`; adding UI fields there couples the clients. |
| D5 | A UI-level job coordinator sits in front of `JobExecutor`. | `JobExecutor::submit` rejects with `SubmitError::Full` (bounded capacity) and `SubmitError::Conflict` (two sends to one device, or overlapping destinations). Sending several selected books therefore needs a UI-side queue that submits the next job when the previous finishes. |
| D6 | Converted files go to a user-configured **export folder** outside the library. | Conversion workspaces are temporary and removed on exit. "Converted" must mean a file the user can find. Needs a Settings row (design gap G3). |
| D7 | Update logic is a pure `update(state, message)` function. All core calls go through a `Services` trait with a fake for tests. | Makes behaviour testable without a window, a library, or a reader. |
| D8 | Dark theme only; palette is a token struct so a light theme can be added later. | Matches the agreed design; keeps scope small. |

## Design gaps (not in the mockup; resolve before the item that needs them)

| № | Gap | Needed by | Proposal |
|---|---|---|---|
| G1 | First-run / library selection screen. | №5 | Centered card: "Choose your Calibre library", path field, Browse button, validation message (uses `config::sanitize_path` and `is_calibre_library`). |
| G2 | Error and notice display, and scan warnings. | №11 | Toast area bottom-right above the bottom bar; Home header shows "N warnings" which opens a list. |
| G3 | Export folder row in Settings, and the first-convert prompt. | №12 | Settings row "Export folder"; if unset, Convert asks for a folder once. |
| G4 | "Choose reader…" UI and manual reader path. | №10 | Popover listing detected readers, plus "Select folder manually…" (core already has the manual-selection API). |
| G5 | Empty library / zero scan results. | №7 | Centered message with "Rescan" and "Change library". |
| G6 | Rescan control. | №7 | A refresh icon button in the Home header. |

## Dependencies planned

| Crate | Purpose | Note |
|---|---|---|
| `iced` 0.14 | UI toolkit. Features to enable: `image`, `svg`, `tokio`, `advanced`, `lazy`. | Confirm the exact feature set in №1 against 0.14 docs. |
| `window-vibrancy` 0.8 | Mica/Acrylic/macOS vibrancy. | Only after the №2 spike is positive. |
| `image` | Decode and resize covers, pre-blur the tint. | In the UI crate only; the core stays free of image dependencies. |
| `rfd` | Native folder picker. | Async API. |
| `rulibre-core` | All business logic. | Already in the workspace. |

---

## Item №1. Create the `rulibre-ui` crate and build it in CI

### What and why

There is no iced code yet. Everything later depends on a crate that builds on Windows, Linux and macOS and is checked by CI from the start, so platform problems appear early rather than at the end.

### How

- Add `crates/rulibre-ui` (binary `rulibre-ui`) to workspace `members`. Do **not** add it to `default-members` until №17, so existing `cargo test`/`check` runs are unaffected while it is unfinished; CI builds it explicitly with `-p rulibre-ui`.
- Dependencies: `rulibre-core` (path), `iced` 0.14 with the features listed above. Use workspace `edition`, `license`, `version`. Keep `publish = false`.
- Create the module skeleton and leave it mostly empty:
  `main.rs`, `app.rs`, `screens/`, `widgets/`, `theme.rs`, `icons.rs`, `catalog.rs`, `selection.rs`, `settings.rs`, `services/`.
- `main.rs` opens one window (title "Rulibre", default 1100×700, minimum 760×520, native decorations) showing a placeholder view. Initialise `env_logger` or `tracing` so later items can log.
- Extend `.github/workflows/readiness.yml` (or add `ui.yml`) with a matrix job for Windows, Ubuntu 24.04 and macOS: `cargo build -p rulibre-ui --locked`, `cargo clippy -p rulibre-ui -- -D warnings`, `cargo test -p rulibre-ui --locked`, `cargo fmt --check`. Install the Linux system libraries iced/wgpu need (verify the list on the runner: Wayland/X11, xkbcommon, Vulkan loader).
- Add `scripts/check-ui.ps1` that runs the same commands locally.

**Completion criterion:** `cargo run -p rulibre-ui` opens an empty window on the developer machine; the CI job builds the crate on all three systems (a CI run is required, not only a local build).

### Status

not done

### Post-implementation notes

—

---

## Item №2. Technical spike: transparency, blur, text, images, icons

### What and why

The design depends on four things we have not proven with iced: a transparent window with OS blur, correct rendering of Cyrillic and Polish text, image display with many covers, and SVG icons. Finding out that one does not work after the screens are built is expensive. This item is a time-boxed spike whose output is a go/no-go note per feature and, if needed, a design adjustment.

### How

Write throwaway code in `crates/rulibre-ui/examples/spike_*.rs` (not shipped):

1. **Window blur.** Create a window with `transparent: true`. Use `window-vibrancy` (Mica/Acrylic on Windows 11, `NSVisualEffectView` on macOS) through the raw window handle that iced exposes (check the 0.14 API for obtaining the handle, e.g. `window::run`). Test with a semi-transparent root container. Record: does it work, can it be toggled at runtime or only at window creation, what happens on Windows 10, what is the fallback on Linux (flat translucent fill).
2. **Text.** Render Cyrillic ("Математика. Утрата определённости"), Polish ("Harry Potter i kamień filozoficzny") and mixed text. Check truncation with ellipsis (iced has no built-in text ellipsis; confirm what we must do ourselves) and font fallback.
3. **Images.** Load 200 JPEG covers into an iced `image::Handle` grid and measure frame time and memory. This feeds the grid design in №9.
4. **Icons.** Render embedded SVG icons (Lucide, ISC licence, copy the licence text into the repo) with the `svg` widget in the theme's text colour.
5. **Modifiers and clicks.** Confirm how to read Ctrl/Shift at click time (iced mouse events do not carry modifiers; the app must track `keyboard::Event::ModifiersChanged` through a subscription).

Output: `docs/UI_SPIKE.md` with the findings and a go/no-go per feature. If window blur is no-go, the Settings toggle and №14 are reduced to cover tint only, and the design is updated.

**Completion criterion:** `docs/UI_SPIKE.md` exists with measured results for all five points, and the spike examples are either deleted or kept under `examples/` with a note.

### Status

not done

### Post-implementation notes

—

---

## Item №3. Core: cover discovery

### What and why

The cover grid and the details pane need a book's cover image. The core does not expose it today (`cover` is mentioned only in test fixtures). Calibre stores the cover as `cover.jpg` in each book folder. Finding it is file logic that belongs in the core, so the TUI or any future client can reuse it. Decoding and resizing stay in the UI crate to keep image libraries out of the core.

### How

- Add `crates/rulibre-core/src/cover.rs` and export it from `lib.rs`.
- API: `pub fn find_cover(book_dir: &Path) -> Result<Option<CoverFile>, Error>` where `CoverFile { path: PathBuf, modified: SystemTime, len: u64 }`. The modification time and size are for cache invalidation in №9.
- Rules: look for `cover.jpg` case-insensitively; accept only regular files (not directories or links that leave the book folder; follow the same link policy as `scanner`); return `Ok(None)` when absent; return `Err` with path context on access errors. Do not read file contents here.
- Do not change `Book` or `ScanReport`; covers are looked up lazily per visible book.
- Tests with the existing temporary-fixture helpers (`test_support`): present, missing, mixed case, `cover.jpg` as a directory, unreadable file, and a very large file (only metadata is read).
- Add a short section to `docs/` (new `docs/COVERS.md`) with the contract.

**Completion criterion:** new tests pass; existing 97 core tests still pass; `scan_metadata` and the TUI still build.

### Status

not done

### Post-implementation notes

—

---

## Item №4. Application architecture, settings persistence, job coordinator

### What and why

The UI will run scans, metadata reads, conversions and transfers in the background and receive devices events. If this plumbing is improvised inside screens the result is untestable. This item builds the skeleton once: state, messages, the pure update function, a service layer over the core, preferences storage, and the job coordinator required by decision D5.

### How

**State and messages.** `app.rs` holds `State` and `Message`. Screens: `Screen::{Setup, Home, Search, Jobs, Settings}` plus `Home { filter: Filter }` for the Library views. `update(&mut State, Message, &dyn Services) -> iced::Task<Message>` contains all logic; views only render.

**Services trait.** `services/mod.rs` defines `trait Services` (scan, read metadata, list backends, convert, send, find cover, load/save settings, open folder picker). The real implementation wraps `rulibre-core`; tests use a fake that returns scripted results. No core types leak into view code except the plain data types (`Book`, `Device`, `Metadata`, `JobEvent`).

**Settings.** `settings.rs`: `UiSettings { view: Grid|List, sidebar_visible, window_blur, cover_tint, prefer_kepub, export_folder: Option<PathBuf> }` stored in `ui.toml` next to the core config (`dirs`). Reuse the atomic write used by `config::save_to` (temporary file, `sync_all`, replace). That logic is private (`Config::save_with`), so extract it into a small public generic `config::write_atomic` in the core (with the existing tests moved or kept green) rather than duplicating it in the UI crate. Missing file means defaults; a corrupted file is reported (not silently replaced) and defaults are used for the session.

**Job coordinator.** `services/jobs.rs`:
- Owns one `JobExecutor`. A UI-side FIFO queue holds requests not yet accepted by the executor.
- Submit rule: try `submit`; on `SubmitError::Full` or `Conflict` keep the request in the UI queue with state *Waiting* and retry when any job finishes. `Closed` and `InvalidDestination` fail the job with the error text.
- Cancelling a Waiting job removes it from the UI queue; cancelling a Running job calls `JobHandle::cancel`.
- Polling: a single `iced::time::every` subscription active only while jobs exist (about 100 ms) calls `try_result` and drains `events` for each handle. This is the same approach as the existing `rulibre-gui/src/job_service.rs`, minus the IPC.
- Maps core `JobEvent` (`State`, `Stage`, `CopyProgress`) into a `JobView { id, title, kind, state, progress: Option<(done,total)>, stage }`. For conversion, report the stage only, never an invented percentage (as in [docs/BACKGROUND_JOBS.md](docs/BACKGROUND_JOBS.md)).
- On exit: cancel everything and call `shutdown` (joins workers) before the process ends; dropping conversion workspaces after jobs stop.

**Stale-response guard.** Port the idea of `rulibre-gui/src/response_guard.rs` (accept a response only if request generation and `BookId` still match) into a small tested helper used for metadata and cover responses.

**Tests.** Pure `update` tests with the fake services; coordinator tests: full queue, conflict then retry, cancel waiting, cancel running, finish ordering, shutdown with queued jobs; settings round-trip, corrupted file, missing file.

**Completion criterion:** all above tests pass; no view code calls the core directly; the coordinator never loses or double-submits a request in the tests.

### Status

not done

### Post-implementation notes

—

---

## Item №5. First run, library selection, scanning

### What and why

Nothing can be shown until a library is chosen and scanned. The old clients had a setup screen; the mockup does not (gap G1). The scan can take seconds on large libraries and must not freeze the window. Scan errors must be visible, not fatal, as already guaranteed by the core (`scan_library` returns `Result<ScanReport, Error>`, warnings are separate).

### How

- Update `docs/design/mockup.html` with the Setup screen from G1 and get sign-off.
- On start: `Config::load()`. `Ok(None)` or an invalid saved library → `Screen::Setup`. A corrupted config shows the error with the config path and offers to choose a library again.
- Setup screen: path text field plus Browse (async `rfd` folder dialog). Validate with `sanitize_path` and `is_calibre_library`; show the core error text mapped to a friendly sentence. On Continue save the config (`Config::save`) and go to Home.
- Scan: submit `JobRequest::Scan { library }` through the coordinator; store `ScanReport.books` in `State` and keep `warnings` for G2. While scanning show a loading state in the content area (spinner or text; Jobs shows the scan job as well). Scan is cancellable (cancel on window close or library change).
- Rescan and change-library actions are defined here as messages; their buttons appear in №7 and №12.
- Tests: first run, valid saved library, saved library deleted, corrupted config, scan success with warnings, scan failure (root unreadable), cancel during scan, library switch while scanning (old result ignored).

**Completion criterion:** with a test library the app starts, shows the Setup screen when no config exists, scans in the background, and the window stays responsive during a scan of a synthetic 20,000-book library.

### Status

not done

### Post-implementation notes

—

---

## Item №6. Theme, icons, and shell layout

### What and why

The whole design is one shell: toolbar, sidebar, content area, details pane, bottom bar. Building it once, with fixed proportions and tokens, prevents each screen from inventing its own spacing and colours.

### How

- `theme.rs`: `Palette` token struct (background, panel, line, hover, selection, text primary/secondary/muted, accent, ok, warning, error) with values from the mockup's CSS variables; build an iced `Theme::custom` and style functions for container, button, text input, scrollable, checkbox. Alpha values prepared for the window-blur variants (used in №14). Spacing and radius constants.
- `icons.rs`: embedded SVG icons as `const` byte slices (home, search, books, list-check, settings, chevron-right/down, layout-sidebar, layout-sidebar-right, list, grid, x, refresh), rendered with a helper that tints them with the theme text colour.
- Shell (`view` in `app.rs`): top toolbar row (sidebar toggle, app name, details toggle), main row (sidebar 148 px, content flexible, details pane 226 px), bottom bar 46 px. Sidebar hidden means removed from the layout, not zero-width. The details toggle is dimmed and inactive on screens that have no details pane (Search, Jobs, Settings).
- Minimum window size keeps the content area usable with both side panes open (check at 760 px wide).
- Static content only in this item (placeholders), so layout can be compared with the mockup side by side.

**Completion criterion:** screenshots of the shell at 760×520 and 1100×700 match the mockup's proportions (manual comparison, recorded in the notes); toggles work; no hard-coded colours outside `theme.rs`.

### Status

not done

### Post-implementation notes

—

---

## Item №7. Catalog, selection, and the Home list

### What and why

The list is the main screen and the most performance-sensitive one. iced has no built-in virtualised list, so rendering thousands of rows naively will be slow. Selection (single, Ctrl, Shift) is also shared by every later screen, so it must be a tested standalone module.

### How

- `catalog.rs`: turns `Vec<Book>` into a view-model: display title, author, format labels (`Format::label`), sort key. Default sort: author, then title, case-insensitive with Unicode-aware comparison. Keep a stable index by `BookId`.
- `selection.rs`: `Selection { selected: BTreeSet<BookId> or Vec, anchor: Option<BookId> }` with `click(id, Modifiers, visible_order)`, `select_all`, `clear`, `move_cursor(delta)`. Semantics identical to the mockup: plain click selects one and sets the anchor; Ctrl/Cmd toggles and moves the anchor; Shift selects the range between anchor and clicked item in the **currently visible order**. Selection is by `BookId`, never by index, so it survives filtering and rescans (ids that disappear are dropped).
- Modifier tracking: subscription on `keyboard::Event::ModifiersChanged` stores current modifiers in `State`; clicks read them (see №2).
- Virtualised list: fixed row height (36 px). Compute the visible slice from the scroll offset (`scrollable::on_scroll` provides the viewport) and render only those rows plus top and bottom spacer containers. Row: thumbnail placeholder column, title, author, format tags.
- Home header: title, book count, list/grid toggle (grid placeholder until №9), rescan button (G6), and "N warnings" opener (G2) when the scan produced warnings. Empty library state (G5).
- Keyboard: Up/Down move the cursor, Shift+Up/Down extend, Ctrl+A select all, Esc clears. Details pane opens on selection (placeholder until №12).
- Tests: selection semantics (all combinations, filtering interaction, anchor after removal), sort order for Cyrillic/Polish/Latin mix, visible-slice computation at boundaries, 20,000-book catalog build time (assert a generous bound, log the measured value).

**Completion criterion:** a 20,000-book library scrolls without visible stutter on the developer machine (record the measured frame time in the notes); selection tests pass.

### Status

not done

### Post-implementation notes

—

---

## Item №8. Sidebar navigation, Library facets, Search screen

### What and why

Navigation is how the user reaches subsets of the library. The design puts search on its own screen and formats/authors under a Library dropdown. Counts and filters come from the already scanned data, so no core changes are needed.

### How

- Sidebar widget: Home, Search, Library (expandable) containing Formats (expandable, each format with a count) and Authors (expandable); then Jobs (badge = unfinished jobs from the coordinator) and Settings. Expansion state is stored in `State` (and optionally in `ui.toml`).
- Facets: computed once per scan in `catalog.rs`: `formats: BTreeMap<Format, usize>`, `authors: BTreeMap<String, usize>` (authors grouped with the same normalisation as sorting). Recompute on rescan only.
- Selecting a facet sets `Screen::Home { filter }`; header title "Library · <value>"; the filter is a pure function over the catalog.
- Search screen: query input at the top, focused automatically when the screen opens; empty state "Type to search by title or author"; results as a list (always list, never grid); "N results"; details pane hidden. Matching: case-insensitive substring over title and author with Unicode-aware lowercase; ignore diacritics only if cheap (otherwise record as a limitation). For large libraries run the filter in a background `Task` and discard stale results by query generation (use the guard from №4).
- Leaving Search keeps the query so returning restores the results (decision recorded in the notes if it changes).
- Tests: facet counts, authors with different case, filter + selection interaction, search with Cyrillic and Polish letters, stale-result rejection, navigation state transitions.

**Completion criterion:** all sidebar paths from the mockup work against a real scanned library; search over 20,000 books stays responsive while typing.

### Status

not done

### Post-implementation notes

—

---

## Item №9. Cover pipeline and the grid view

### What and why

The grid view is a required feature. It needs fast, bounded-memory cover loading: decoding full-size JPEGs for every card would exhaust memory and block the UI.

### How

- `services/covers.rs`: pipeline per `BookId`: `cover::find_cover` (№3) → decode → resize to a fixed thumbnail size (about 2× the card size for HiDPI) → store as PNG/JPEG in a disk cache under the OS cache directory (`dirs::cache_dir()/rulibre/covers`), keyed by `BookId` plus the cover file's modification time and size → in-memory LRU of ready iced image handles (bounded count, e.g. 300).
- Loading runs off the UI thread (`Task::perform` with `spawn_blocking`) with a small concurrency limit. Only covers for visible cards (plus one screen of lookahead) are requested; requests for cards that scrolled away are cancelled or ignored (generation check, №4).
- A missing or undecodable cover shows a deterministic placeholder (colour derived from the title hash, title text), as in the mockup. Decode errors are logged once per book, not shown as toasts.
- Virtualised grid: card width fixed (about 96–120 px), columns computed from available width using the `responsive` widget, fixed row height; render only visible rows with spacers, like the list.
- Persist the list/grid choice in `ui.toml`. Selection works identically to the list (same `Selection`). Titles under cards are truncated manually.
- The same cache feeds the list's small thumbnail column and the details pane (№12).
- Tests: cache key invalidation when the file changes, LRU eviction, placeholder determinism, grid column computation at several widths, request cancellation on fast scrolling (using the fake `Services`).

**Completion criterion:** scrolling the grid of a 5,000-book library with real covers keeps memory bounded (record peak working set) and never blocks input; the cache is reused on the second start.

### Status

not done

### Post-implementation notes

—

---

## Item №10. Device bar

### What and why

The bottom bar is always visible and represents the currently connected reader. The core already provides `DeviceWatcher`/`DeviceTracker` with events for connect, disconnect, replace, path change and polling failure ([docs/DEVICE_WATCH.md](docs/DEVICE_WATCH.md)). The UI must only render those states correctly.

### How

- Start `DeviceWatcher::start(interval, publish)` at launch; `publish` forwards `DeviceEvent` into the iced runtime through a subscription backed by a channel (`iced::stream::channel`), returning `false` from `publish` when the receiver is gone so the watcher stops itself. Stop the watcher on exit (it joins its thread).
- Hold the latest snapshot in `State`: devices list, selected id, warnings. Use `DeviceTracker::select` semantics: a single reader is selected automatically, several readers are not.
- Bar states from the mockup: *none* (grey dot, "No reader connected", hint text instead of a button), *one* (green dot, name, mount point, free space, Send button), *several* (amber dot, "N readers found · choose one", "Choose reader…"). `PollFailed` and detection warnings appear as a small warning text with the diagnostic (full text in the toast area, №11).
- Free space: query available bytes for the mount point. The core does not expose free space today (checked: no such function in `volumes`/`transfer`). Add `volumes::free_space(mount_point) -> Result<u64, Error>` to the core (Win32 `GetDiskFreeSpaceExW`, `statvfs` on Unix) with tests on temporary directories and mock failures. Refresh at each poll; a failure shows "free space unknown", not an error.
- "Choose reader…" (G4): a popover listing detected readers (name, label, mount point, filesystem, read-only flag) plus "Select folder manually…" which uses the core manual-selection API ([docs/DEVICE_DETECTION.md](docs/DEVICE_DETECTION.md)) with the same validation. Read-only readers are shown but Send is disabled with an explanation.
- The Send button itself is wired in №11; here it is rendered and enabled/disabled by state.
- Tests with the fake device feed: connect, disconnect, replace, two readers, selection, poll failure, read-only device, watcher shutdown.

**Completion criterion:** plugging a folder fixture through the core's test snapshot support produces all bar states in the right order; no thread remains after exit.

### Status

not done

### Post-implementation notes

—

---

## Item №11. Send, Jobs screen, toasts

### What and why

This is the first end-to-end user action: send selected books to the reader and see progress, queueing and errors. It exercises the coordinator from №4 against the real core and requires the Jobs screen, the bottom-bar progress and an error display (G2).

### How

- "Send to reader" (and "Send N to reader" for multi-selection): for each selected book submit `JobRequest::Send { book, author, device, options }` through the coordinator, one job per book, in selection order. `options` come from `UiSettings.prefer_kepub` (`PreparationOptions { kobo: PreferKepub | Compatible }`). Jobs after the first wait in the UI queue because writes to one device conflict in the executor (D5).
- Book path/author come from the catalog (`Book.path`, `Book.author`). Disable Send when no reader is selected, the reader is read-only, or the selection is empty.
- Bottom bar progress: show the running copy job as `42%` with a progress bar, "· N queued" and Cancel. Progress comes from `JobEvent::CopyProgress { completed, total }`; before copying starts (preparing/converting) show the stage text, not a percentage.
- Jobs screen: sections Running / Up next / Done (last N), per-job Cancel for running and waiting jobs, "Clear finished", empty state. Job rows show title, state/stage/progress and, for finished jobs, the result message or error. Sidebar Jobs badge shows unfinished count.
- Toasts (G2): bottom-right stack above the bar, auto-dismiss after a few seconds for success ("Sent to Kobo Libra 2"), sticky with a close button for errors. Error text is built from `Error`/`ErrorKind` at this boundary only; include the path where it helps; never show raw debug output. Handle the specific cases the core distinguishes: destination conflict (file exists, no silent overwrite), device disappeared (`PartialTransfer` with a leftover `.part` path), permission denied, cancelled.
- Cancel while copying calls `JobHandle::cancel`; cancelled jobs show as Cancelled, not Failed.
- Update the mockup if the toast or job-row appearance differs from what is implemented.
- Tests: queueing of three sends, cancel one waiting and one running, conflict error mapping, device removed mid-transfer, progress mapping, badge counts, toast lifetime.

**Completion criterion:** sending three books to a fixture device (temporary directory classified as a reader through the core test support) queues, shows progress, completes, and leaves the library untouched (snapshot check as in the core integration tests). No claim about a physical reader is made.

### Status

not done

### Post-implementation notes

—

---

## Item №12. Settings screen

### What and why

Several later items read user preferences; the screen that edits them must exist first. It also resolves the export folder (G3) that conversion needs.

### How

- Settings screen rows (as in the mockup): Window blur (toggle), Cover tint (toggle), Prefer KEPUB for Kobo (toggle), Library folder (path + Change…), and the new Export folder (path + Change…, G3). Update the mockup with the Export folder row first.
- Changing the library folder reuses the Setup validation, saves the config, clears selection and cover caches for the old library, and starts a scan. A failed validation keeps the old library.
- Export folder: chosen with the folder picker; must exist, be writable, and be outside the library. The core already enforces this in `converter::validate_output_dir`, which is `pub(crate)`; make it public (or add a thin public wrapper) so the Settings screen shows the same verdict the conversion API will give, instead of re-implementing it. Store in `ui.toml`.
- Every change is saved immediately through the atomic writer; a failed save shows a toast and keeps the previous value in the file.
- Window blur and Cover tint toggles are stored now; their effect is implemented in №14. Until then the toggles have no visible effect and the notes must say so.
- Tests: each setting round-trips; invalid export folder (inside the library, missing, read-only); library change while a scan is running; failed save keeps the old file.

**Completion criterion:** all settings persist across restarts; changing the library works end to end with a second test library.

### Status

not done

### Post-implementation notes

—

---

## Item №13. Details pane, metadata, multi-select, Convert

### What and why

The details pane is the right-hand pane in the design: single-book metadata, a "N selected" summary for several books, and the Convert action for all selected books. It needs asynchronous metadata loading with stale-response protection, because the user can change selection faster than OPF files are read.

### How

- Visibility: the pane is shown on Home/Library screens when the toggle is on; any selection change forces it open (as designed). The toggle persists in `ui.toml`. The pane is never shown on Search, Jobs, Settings.
- Single selection: cover (from the №9 cache, larger size), title, then fields Author, Publisher, Year (from `Metadata.date`), Language, Series (+ index), Formats, Rating, identifiers, tags, description. Hide empty fields. Metadata is requested with `JobRequest::Metadata { book }` or a direct `read_metadata` call in a blocking task; key the response with the №4 guard (request generation + `BookId`), so A→B→A does not show stale data. While loading show the title/author from the catalog immediately (the same fallback as `Metadata::with_fallback`), then fill in.
- Metadata errors: the book still shows with fallback fields; the error appears as a muted line "Metadata could not be read" with the cause available in a tooltip/toast, not a blocking error.
- Description text: the core already returns it normalised (paragraphs preserved); render with wrapping and a maximum height with scroll.
- Multi-selection: "N selected" counter, first five titles and "+ M more", Convert for all.
- Convert menu: request backends once at start (`JobRequest::Backends`) and refresh on demand; compute targets with `converter::target_formats(files, backends)` for the selected book(s). For several books show only targets available for **all** of them and list the skipped books. Routes without a tool show "no route" with the reason (missing, broken, no KEPUB support) from `Backend` diagnostics. Never show a target the core would reject.
- Convert action: for each selected book submit a conversion job (`JobRequest::ConvertFiles` with `library`, `output_dir` = the export folder from №12, `files`, `target`). If no export folder is set, ask once (G3) and store it. Conflicts with an existing output name are reported, never overwritten. Result toast says "Converted · saved to <folder>" with an Open folder action. The converted file is **not** added to the library, and the UI says so only where needed (Jobs row text).
- Conversion progress shows stage only (core does not provide a reliable percentage).
- Tests: stale metadata rejection (A→B→A), fallback on metadata error, multi-selection targets intersection, "no route" reasons, conversion job creation per book, missing export folder flow, name conflict.

**Completion criterion:** converting a fixture book with the fake backend produces the correct job and result; if `ebook-convert` is installed on the developer machine, one real TXT → EPUB conversion is run and recorded (otherwise recorded as not verified).

### Status

not done

### Post-implementation notes

—

---

## Item №14. Visual effects: window blur and cover tint

### What and why

The design makes blur configurable. This item turns the Settings toggles into real effects, based on the №2 spike results. Skipping the unproven parts explicitly is better than shipping flickering or broken transparency.

### How

- **Window blur** (if №2 is go): create the window as transparent; on Windows 11 apply Mica/Acrylic through `window-vibrancy`, on macOS the vibrancy view; on other systems no effect. When the toggle is off or unsupported, use opaque palette colours. If the spike showed it can only be chosen at window creation, the toggle takes effect on the next start and the Settings row says so.
- Translucent palette variants: panels and the root use alpha colours only while blur is active, to keep text contrast; check contrast of primary and secondary text on a bright wallpaper and on a dark one.
- **Cover tint:** for the selected book (single selection only), take its cover, shrink it to a small bitmap, blur it on the CPU once (box or Gaussian blur from `image`), cache the result with the cover cache key, and draw it dimmed behind the details pane content with low opacity. For several books use the accent colour. Compute in a background task; the pane renders without the tint until it is ready.
- Both effects respect their toggles immediately where possible.
- Do not implement per-element backdrop blur (decision above).
- Tests: tint cache key and reuse, tint disabled does no work, palette selection function for blur on/off.
- Manual check on Windows 11 with a bright and a dark wallpaper; record Windows 10, macOS and Linux as verified or not verified.

**Completion criterion:** toggling the effects works as described in Settings on the developer's Windows machine, with screenshots in the notes.

### Status

not done

### Post-implementation notes

—

---

## Item №15. Keyboard, states, accessibility, performance pass

### What and why

The design is lightweight and informative; the details that make it feel finished are keyboard access, honest empty/error states and consistent performance. These were deliberately left out of the screen items so they are applied uniformly.

### How

- Keyboard map: `/` opens Search; arrows/PageUp/PageDown/Home/End move the cursor; Shift extends; Ctrl+A selects all; Enter focuses/opens details; Esc closes popovers, clears selection, or leaves Search; `T` sends the selection; `C` opens the Convert menu; `Ctrl+B` toggles the sidebar; `Ctrl+I` toggles details. Show shortcuts in tooltips. Shortcuts are ignored while a text field has focus (except Esc).
- Focus: visible focus outlines on buttons and rows; logical tab order; popovers close on outside click.
- States: empty library (G5), search with no results, no metadata, missing cover, scan failure with retry, reader read-only, converter missing. Each has text written as an invitation and a next action, in the project's language style (sentence case, no "successfully").
- DPI and scaling: check 100%, 150%, 200% scale on Windows; images stay sharp; text does not clip in the sidebar or rows.
- Performance: profile with a 20,000-book synthetic library: startup to first paint, scan, filter on each keystroke, scrolling list and grid. Fix anything above a budget recorded in the notes (suggested: first paint under 500 ms after scan results arrive, keystroke filter under 16 ms). Make sure there is no per-frame allocation in the visible-slice code.
- Logging: errors that are not shown to the user (cover decode failures, poll warnings) go to the log with context.
- Tests: keyboard handler table tests (key + modifiers + focus context → message), state rendering tests where iced's test tooling allows (see №16).

**Completion criterion:** a written manual checklist (kept in `docs/UI_CHECKLIST.md`) is executed on Windows and its results recorded; performance numbers are in the notes.

### Status

not done

### Post-implementation notes

—

---

## Item №16. Test hardening, CI, documentation

### What and why

Items above add tests as they go; this item closes the gaps, makes CI authoritative for the new crate on all three systems, and documents the UI for contributors and users.

### How

- Evaluate iced's headless test tooling for 0.14 (`iced_test` simulator, if available) and add a few UI-level tests: click a sidebar item → screen changes, Ctrl+click selection, details toggle, Send queues jobs. If the tooling is not usable, record that and rely on update-level tests.
- Integration test that mirrors `docs/READINESS.md`: config → scan → select → send to fixture device → verify library snapshot unchanged, plus a cancel and a conflict case, driven through the real coordinator and the real core (not the fake).
- Add the UI crate to the readiness workflow so Windows, Linux and macOS run its tests; keep logs as artifacts like the existing job. A green remote run is required before marking this item done.
- Documentation: `docs/UI.md` (architecture, module map, message flow, how to add a screen, how to run), update `README.md` (screenshots, how to run, supported platforms, external tools), update `docs/READINESS.md` with the UI matrix. Include an explicit limitations section (no physical reader verification unless done, MTP unsupported, light theme absent, window blur platform notes).
- Add `docs/UI_CHECKLIST.md` results link from №15.

**Completion criterion:** CI is green on the three systems for the UI crate; documentation is complete; every limitation known from items №1–№15 is listed.

### Status

not done

### Post-implementation notes

—

---

## Item №17. Replace the old GUI and prepare the release

### What and why

Two GUI clients would double maintenance and confuse users. After parity, the Tauri/Leptos client should be removed and the iced UI made the default graphical client.

### How

- Parity check against the old GUI: setup, scan, search, details, convert, send, device watching, cancel. Record anything the old GUI did that the new one does not (and decide to implement, drop, or move to a backlog).
- Remove `crates/rulibre-gui` and its pieces: `Trunk.toml`, `tauri.conf.json`, `gen/`, `capabilities/`, `index.html`, `style.css`, Leptos/WASM code, `scripts/check-wasm-ipc.cjs`, the WASM and Tauri steps in `scripts/check-clients.ps1`/`check-readiness.ps1` and in CI (wasm target, `wasm-bindgen-cli`, Node setup, WebKit packages). Delete the `*_model.rs` `#[path]` sharing only if nothing else uses it (check the TUI and tests first); otherwise leave the modules where they are.
- Add `rulibre-ui` to `default-members`; update workspace `description`/`keywords` if needed; decide the binary name users run (suggested: keep `rulibre` for the TUI and `rulibre-gui` as the iced binary name via `[[bin]] name`, so existing instructions stay valid).
- Windows: application icon and version info embedded in the executable (build script, e.g. `winresource`), no console window in release builds (`windows_subsystem = "windows"`), and a release build check.
- Update README, docs links, and the crates.io metadata; remove references to Tauri/Trunk.
- Tag the repository after the final run of the full checks.

**Completion criterion:** a clean clone builds with `cargo build --release`, the full CI matrix is green without WASM/Tauri jobs, the README describes only existing features, and the release binary starts and scans a test library on Windows.

### Status

not done

### Post-implementation notes

—

---

## Out of scope for this plan

- Writing to Calibre libraries or `metadata.db`; editing metadata.
- MTP devices, network devices, wireless transfer.
- Light theme, custom title bar, localisation.
- Series/tag/language filters (the data exists in OPF but is not part of the scan snapshot today).
- Drag and drop, multi-select without a keyboard (checkboxes), reading books in the app.
- Installers and auto-update.
- Per-element backdrop blur.
