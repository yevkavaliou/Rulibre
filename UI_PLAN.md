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

- **Appearance amendment approved after №13:** Settings includes Text size (14/16/18/20 px, default 16) and Cover size (120/160/200 px wide, default 160). The right-pane divider is draggable (240–480 px, default 300, constrained by the window and center content); its width persists in ui.toml. Convert distinguishes active routes with light text on blue buttons from readable grey no-route entries. See [docs/UI_APPEARANCE.md](docs/UI_APPEARANCE.md). These were not explicitly included in №14–№17 and were implemented as corrections to the existing screens.

- **Three panes plus a fixed bottom bar.** Left sidebar (hideable, fully removed from layout when hidden), centre content, right details pane (hideable via the toggle at the top right). The bottom bar can never be hidden.
- **Sidebar items:** Home, Search, Library (dropdown containing Formats and Authors dropdowns), Jobs (badge with the number of unfinished jobs), Settings. Home is the default screen and shows all books.
- **Home / Library filter screens:** list or cover grid (user toggle, grid is default in the mockup). Selecting a Format or Author item shows the same layout filtered, titled "Library · EPUB".
- **Search** is a separate screen: search field at the top, results as a list below, details pane hidden. Nothing is shown until the user types. Filtering is live.
- **Details pane:** shows the selected book. Selecting a book always opens the pane, even if the user had hidden it. With several books selected it shows "N selected", a short title list and Convert for all of them. Multi-select is keyboard-only for now (Ctrl+click toggles, Shift+click selects a range).
- **Convert** lives in the details pane, not in the bottom bar. Routes that have no tool are shown as "no route".
- **Bottom bar** shows only the connected reader: none / one / several (then the user must choose). It shows free space, transfer progress with Cancel, and "Send to reader" (or "Send N to reader" for a multi-selection, queued one after another).
- **Jobs** is a screen: Running, Up next, Done, per-job Cancel, "Clear finished".
- **Settings** is a screen: window opacity (slider), blur window (toggle), prefer KEPUB for Kobo, library folder, export folder, text and cover size. See [docs/UI_BLUR.md](docs/UI_BLUR.md).
- **Visual effects (configurable):** a translucent window (opacity slider) optionally over a blurred desktop (Windows acrylic, "Blur window" toggle), modelled on Sonora. Revised in №14: the №2 "no-go" only applied to wgpu's default surface; see [docs/UI_BLUR.md](docs/UI_BLUR.md). Per-element backdrop blur (Sonora's GPUI feature) is explicitly **not** planned: iced has no equivalent. The cover-colour wash first tried for this item was dropped.
- Dark theme only in the first version.

## Decisions taken (recorded so they are not reopened by accident)

| № | Decision | Why |
|---|---|---|
| D1 | Use **iced 0.14** (crates.io), not GPUI. | Stable released crate, documented, already named in the original plan. GPUI is an unreleased git dependency with a fork to maintain. |
| D2 | New crate `crates/rulibre-ui`. The Tauri/Leptos `rulibre-gui` stays until the new UI reaches parity and is removed in №17. The TUI (`rulibre`) stays. | Keeps the repository buildable and gives a fallback during development. |
| D3 | Native window decorations (revised in №14: Windows with window blur available uses the toolbar as its own title bar); the toolbar in the mockup is an in-app row below the OS title bar. | A custom title bar needs drag, resize, snap and DPI handling per OS. Not worth it for v1. |
| D4 | UI preferences live in a separate `ui.toml`, not in core `Config`. | The TUI shares core `Config`; adding UI fields there couples the clients. |
| D5 | A UI-level job coordinator sits in front of `JobExecutor`. | `JobExecutor::submit` rejects with `SubmitError::Full` (bounded capacity) and `SubmitError::Conflict` (two sends to one device, or overlapping destinations). Sending several selected books therefore needs a UI-side queue that submits the next job when the previous finishes. |
| D6 | Converted files go to a user-configured **export folder** outside the library. | Conversion workspaces are temporary and removed on exit. "Converted" must mean a file the user can find. Needs a Settings row (design gap G3). |
| D7 | Update logic is a pure `update(state, message)` function. All core calls go through a `Services` trait with a fake for tests. | Makes behaviour testable without a window, a library, or a reader. |
| D8 | Dark theme only; palette is a token struct so a light theme can be added later. | Matches the agreed design; keeps scope small. |
| D9 | Window blur ships on Windows through a DirectComposition swap chain (revised in №14; originally "no blur in v1"). Two marked patches to iced live in `vendor/`. Other platforms stay opaque. | The №2 spike only saw wgpu's default `Opaque` surface. Presenting through `DxgiFromVisual` with premultiplied alpha and no redirection bitmap works; see [docs/UI_BLUR.md](docs/UI_BLUR.md). |

## Design gaps (not in the mockup; resolve before the item that needs them)

| № | Gap | Needed by | Proposal |
|---|---|---|---|
| G1 | First-run / library selection screen. | №5 | Centered card: "Choose your Calibre library", path field, Browse button, validation message (uses `config::sanitize_path` and `is_calibre_library`). |
| G2 | Error and notice display, and scan warnings. | №11 | Toast area bottom-right above the bottom bar (approved and implemented in №11); Home header shows "N warnings" which opens a list. |
| G3 | Export folder row in Settings, and the first-convert prompt. | №12 | Settings row "Export folder"; if unset, Convert asks for a folder once. |
| G4 | "Choose reader…" UI and manual reader path. | №10 | Popover listing detected readers, plus "Select folder manually…" (core already has the manual-selection API). |
| G5 | Empty library / zero scan results. | №7 | Centered message with "Rescan" and "Change library". |
| G6 | Rescan control. | №7 | A refresh icon button in the Home header. |

## Dependencies planned

| Crate | Purpose | Note |
|---|---|---|
| `iced` 0.14 | UI toolkit. Features to enable: `image`, `svg`, `tokio`, `advanced`, `lazy`. | Confirm the exact feature set in №1 against 0.14 docs. |
| `window-vibrancy` 0.8 | Acrylic backdrop for window blur. | Windows-only normal dependency; dev dependency elsewhere (macOS lab). |
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

done

### Post-implementation notes

- 12 October 2026: added `crates/rulibre-ui` to workspace members only, with the planned module skeleton, inherited package metadata, `publish = false`, core dependency, iced 0.14 and env_logger. The five requested features were confirmed against https://docs.rs/crate/iced/0.14.0/features; default renderer/window-system features are retained.
- Startup uses native decorations, title "Rulibre", size 1100×700, minimum 760×520, dark theme and a centered placeholder. No library or device operations are implemented.
- Added `.github/workflows/ui.yml`: Windows, Ubuntu 24.04 and macOS run `scripts/check-ui.ps1`, with Linux Wayland/X11, xkbcommon and Vulkan packages and uploaded logs. Linux package installation and the locked build passed on the Ubuntu 24.04 runner.
- Local Windows checks passed: `cargo fmt --all -- --check`, `cargo clippy --locked --all-targets -- -D warnings`, `cargo test --locked`, and `scripts/check-ui.ps1` (locked UI build, Clippy with all targets, UI tests, workspace formatting). UI currently has no behavioural tests because this item adds only a placeholder; default workspace has 100 passing tests and 3 ignored tests.
- `cargo run -p rulibre-ui --locked` was executed on Windows: process 7484 created an OS window with title "Rulibre", a nonzero window handle and `Responding = True`. Visual layout and minimum-size resizing have not been manually verified. Startup logged a malformed system font warning for `mstmc.ttf`; it did not prevent window creation.
- First remote run: https://github.com/yevkavaliou/Rulibre/actions/runs/38231733338. Windows and macOS UI checks passed; Linux exposed an existing core `collapsible_if` warning. The existing readiness run also exposed a macOS fixture trying to create an invalid UTF-8 filename (rejected by the filesystem). Collapsed the Linux conditional without changing behaviour and excluded macOS from that filesystem-specific test; it remains enabled on other Unix platforms. Repeated all local checks above successfully after these fixes.
- Remote UI CI rerun passed on Windows, Ubuntu 24.04 and macOS: https://github.com/yevkavaliou/Rulibre/actions/runs/38232451939 (tested implementation commit `6314cc3`). Each runner completed the locked build, Clippy, tests and workspace formatting checks. The separate, broader readiness rerun is https://github.com/yevkavaliou/Rulibre/actions/runs/38232452025; it was still running when the UI matrix completed. No physical reader verification was performed. Implementation branch: `codex/ui-item-1`; initial commit: `bd3f42b`; platform fixes: `6314cc3`. No PR was created.

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

done

### Post-implementation notes

- 12 October 2026: retained `examples/spike_capabilities.rs` and its README, added dev-only `image` and `window-vibrancy` 0.8.1, embedded Lucide SVG with the original license, and added `scripts/measure-ui-spike.ps1`. Findings, go/no-go decisions, exact measurements and raw evidence are in [docs/UI_SPIKE.md](docs/UI_SPIKE.md).
- Ran the lab on Windows 11 build 26100 at scale 1 with Vulkan, DX12 and tiny-skia. All Mica/off/Acrylic/off API calls succeeded on the same window. Vulkan and DX12 support only opaque surface composition in the tested configuration, so OS window blur is no-go for v1. Removed its mockup toggle and reduced the planned settings/№14 work to cover tint. This is not a universal claim about every possible iced renderer setup.
- Inspected renderer screenshots: exact Cyrillic and Polish titles, mixed-script font fallback, 200 synthetic JPEGs and the theme-coloured SVG render. A fixed-width unwrapped title overflows: real width-aware ellipsis remains work for №7. Debug JPEG decode + 200 RGBA handles took 6.74–6.78 s; Vulkan/DX12 redraw interval means were 8.2–8.5 ms; software was 270–280 ms. See the report for memory, p95 and measurement limitations.
- Checks passed locally: workspace formatting, default workspace locked Clippy (all targets) and tests, and `scripts/check-ui.ps1` including the example's modifier/focus-loss test. CI now includes example tests through the same script. No new commit, PR or remote CI run was created.
- Remaining limitations: renderer screenshots do not show native desktop blur; physical Ctrl/Shift clicks, Windows 10, macOS, Linux, higher DPI and release performance were not verified. The experiment uses synthetic cover fixtures and no physical reader or Calibre library. Main application behaviour remains the №1 placeholder. No work on №3 was started.

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

done

### Post-implementation notes

- 12 October 2026: added public `cover::find_cover` and `CoverFile { path, modified, len }` in `rulibre-core`, with no image dependency or changes to `Book`/`ScanReport`. Discovery matches `cover.jpg` case-insensitively, reads directory entries and metadata only, ignores directories and links resolving outside the canonical book folder, and returns contextual I/O errors. The returned path is canonical. Multiple case variants use deterministic native path order.
- The scanner currently follows links without checking containment; cover discovery follows the explicit stricter requirement in this item rather than copying that gap. Internal links are accepted; broken links return an error. The contract and the concurrent-path-change limitation are documented in [docs/COVERS.md](docs/COVERS.md).
- Added seven tests exercised on Windows: present cover with library byte snapshot, absent/directory/unrelated filename, mixed case, 512 MiB metadata-only discovery, injected metadata permission denial, missing book directory, and a file locked against content reads whose metadata remains accessible. Unix additionally has link containment/broken-link and no-read-permission tests; those Unix-specific tests were not run locally.
- Final local checks passed: `cargo fmt --all -- --check`, `cargo clippy --locked --all-targets -- -D warnings`, `cargo test --locked` (107 passed across the default workspace, 3 ignored; core accounts for 104 passed), `scripts/check-ui.ps1` (locked UI build, all-target Clippy/tests, formatting; one example test passed), `cargo build --locked -p rulibre`, and `cargo build --locked -p rulibre-core --bin scan_metadata`. An initial sandbox run failed because Windows sandbox denied `canonicalize` on temporary fixtures; final checks ran outside that sandbox. An initial Windows test incorrectly expected a content lock to deny metadata access; it was corrected to verify that discovery works without reading contents, then the full checks passed.
- No commit, PR or remote CI run was created. Tests use synthetic fixtures only; no physical reader or personal Calibre library was verified. №4 has not been started.

---

## Item №4. Application architecture, settings persistence, job coordinator

### What and why

The UI will run scans, metadata reads, conversions and transfers in the background and receive devices events. If this plumbing is improvised inside screens the result is untestable. This item builds the skeleton once: state, messages, the pure update function, a service layer over the core, preferences storage, and the job coordinator required by decision D5.

### How

**State and messages.** `app.rs` holds `State` and `Message`. Screens: `Screen::{Setup, Home, Search, Jobs, Settings}` plus `Home { filter: Filter }` for the Library views. `update(&mut State, Message, &dyn Services) -> iced::Task<Message>` contains all logic; views only render.

**Services trait.** `services/mod.rs` defines `trait Services` (scan, read metadata, list backends, convert, send, find cover, load/save settings, open folder picker). The real implementation wraps `rulibre-core`; tests use a fake that returns scripted results. No core types leak into view code except the plain data types (`Book`, `Device`, `Metadata`, `JobEvent`).

**Settings.** `settings.rs`: `UiSettings { view: Grid|List, sidebar_visible, cover_tint, prefer_kepub, export_folder: Option<PathBuf> }` stored in `ui.toml` next to the core config (`dirs`). Reuse the atomic write used by `config::save_to` (temporary file, `sync_all`, replace). That logic is private (`Config::save_with`), so extract it into a small public generic `config::write_atomic` in the core (with the existing tests moved or kept green) rather than duplicating it in the UI crate. Missing file means defaults; a corrupted file is reported (not silently replaced) and defaults are used for the session.

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

done

### Post-implementation notes

- 12 October 2026: introduced the UI library boundary, `State`, `Screen`/`Filter`, messages and `update(state, message, services) -> Task<Message>`. The binary injects `RealServices`; views remain the item №1 placeholder and call no core services. No new screen or mockup change was needed. Startup loads UI preferences asynchronously; the shell/setup and library startup remain work for №5–№6.
- Added `Services` with core-backed scan, metadata, backend discovery, conversion and send requests, background cover lookup and settings IO, and the native asynchronous folder picker (`rfd` 0.17.2). Blocking file operations use Tokio's blocking pool. Test services and executor/handle scripts require no window or reader.
- Added `UiSettings` in `ui.toml` beside `config.toml`, with Grid/List, sidebar/details visibility, cover tint, Kobo preference and optional export folder. Details visibility is included for the later pane-persistence requirement. Missing files use defaults; corrupt files produce a session notice and remain untouched. Extracted public `config::write_atomic` from the existing writer, retaining temporary-file creation, sync, replacement and cleanup. Saves are serialized/coalesced through update, and a failed save keeps the previously committed settings.
- Added a FIFO job coordinator owning one core executor, stable UI job IDs independent of core acceptance IDs, Full/Conflict retries, permanent refusal diagnostics, waiting/running cancellation, event/stage/real-copy-progress mapping and exactly-once result collection. Conversion exposes stage only. The 100 ms subscription runs only while unfinished jobs exist. Terminal events arriving before their result keep polling active. Close cancels jobs and joins workers before dropping queued/running/retained conversion workspaces; shutdown is also idempotent on runtime return/drop. `JobRequest` now derives Clone/Debug so refusal cannot consume the retained request.
- Metadata and cover responses use independent generation + BookId guards, including A → B → A and invalidation when selection/details leave the active screen. Results are data in State for later screens; notices/toasts and the visible Jobs screen are not implemented in this item.
- Local Windows checks passed: `scripts/check-ui.ps1` (workspace formatting, locked UI build, all-target UI Clippy and tests: 23 library tests plus the existing spike example test), `cargo clippy --locked --all-targets -- -D warnings`, `cargo test --locked` (107 passed in the default workspace, 3 ignored), and `git diff --check`. Tests cover queue capacity/conflict scripts, cancellation, completion ordering, permanent refusal, terminal-event race, progress, shutdown/workspace ordering, stale responses, settings round-trip/corruption/missing file/failed replacement, serialized saves, and actual background settings/cover IO and executor failure on temporary fixtures. Final checks ran outside the Windows sandbox because it denies path canonicalization/network access needed by Cargo and fixtures.
- No commit, PR or remote CI run was created. Native folder-picker interaction and window-close behaviour were not manually verified; platform checks were local Windows only. No physical reader or personal Calibre library was used. The view remains a placeholder; cover tint has no visual effect yet, and notices are stored for the later error UI. №5 was not started.

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

done

### Post-implementation notes

- 12 October 2026: added G1 Setup and simulated scan states to `docs/design/mockup.html`; the user approved the design before implementation. The production screen is native iced 0.14. Startup asynchronously loads core config, validates the saved library and opens Setup for missing/invalid/corrupted config; corrupt-config errors include its path. Setup has a path field, async native Browse, validation through core `sanitize_path`/`is_calibre_library`, and atomic config save before Home/scanning. IO lives in Services; failed validation/save keeps the prior config/library.
- Scans run through the existing coordinator. State stores books, warning diagnostics, active scan identity and loading/error/cancelled states. Home and a minimal scan Jobs view show cancellation; failures offer retry or choosing another library. Library changes cancel scans, generation guards reject stale config/picker/validation replies, and active job identity rejects old scan results. Obsolete scans are not retained in generic result history. Shutdown cancels/joins workers. Shell/palette and full Jobs/device/warning UI remain their later items.
- Checks passed locally on Windows: `scripts/check-ui.ps1` (fmt, locked build, all-target Clippy and tests: 30 UI library tests plus one existing spike test), default-workspace locked all-target Clippy and tests (107 passed, 3 ignored), and `git diff --check`. New tests cover startup/config errors, validation/save failure, warnings, scan failure, cancellation, switching libraries, stale replies and close during scan. RealServices tests exercise real config/scanner/coordinator IO on temporary fixtures and check source contents remain unchanged. Cargo/fixture checks ran outside the Windows sandbox because it blocks path access.
- Executed `cargo run -p rulibre-ui --example setup_scan_probe --locked` with a real iced window and synthetic 20,000-book library using isolated temporary configuration. Final first-run scan: 2,418 ms, 150 heartbeat events, maximum update gap 24 ms; saved-config scan: 2,218 ms, 138 events, maximum gap 25 ms. Setup and scanning reached the real view; renderer screenshots were inspected and transition title clipping fixed. Report, raw output, screenshots and reproduction instructions: [docs/UI_SETUP.md](docs/UI_SETUP.md).
- Limitations: the fixture `metadata.db` is only a validation marker, not SQLite. Heartbeats/view calls measure event-loop activity, not GPU frames or physical input latency. Native Browse interaction, higher DPI, minimum-size layout, Linux/macOS and physical-reader/personal-library use were not verified. No commit, PR or remote CI run was created. №6 was not started.

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

done

### Post-implementation notes

- 12 October 2026: implemented dark Palette tokens from the mockup, custom iced theme, spacing/radius constants and styles for containers, buttons, text inputs, scrollables and checkboxes. Background/panel stay opaque (D9); overlay alpha values are preserved. The missing mockup error token currently aliases warning. All production UI colour literals are confined to `theme.rs`.
- Added thirteen embedded, tintable SVG icons and ISC attribution. The app view now uses a shared shell: 34 px toolbar, 148 px sidebar, flexible centre, 226 px details and fixed 46 px bottom bar. Hidden panes are removed from the row. Setup and existing scan/cancel/retry flows are retained; other content remains placeholders. Home/Search/Jobs/Settings navigation is available to inspect the shell; Library facets and list/grid actions remain later items.
- Sidebar/details toggles update immediately and persist through the existing Services writer. Rapid combined edits coalesce; a failed save restores the last saved preferences and records a notice. Details are dimmed/inactive on Search, Jobs, Settings and Setup. Added tests for these transitions, queued toggles, failed-save rollback and the 386 px centre at minimum width.
- Executed a real iced shell probe with isolated temporary preferences and RealServices. Captured and inspected 1100×700 and 760×520 at scale 1, hidden-pane states, Search/Jobs/Settings and Setup. Compared shell proportions/colours with the current mockup CSS and its saved browser Settings screenshot; no clipping in the inspected minimum-size shell. Fresh browser rendering of the local HTML was blocked by the browser's file-URL policy. Screenshots, raw probe output, comparison details and reproduction instructions: [docs/UI_SHELL.md](docs/UI_SHELL.md).
- Checks passed on Windows: `scripts/check-ui.ps1` (fmt, locked build, all-target UI Clippy/tests: 32 library tests plus 1 spike test), default-workspace locked all-target Clippy and tests (all executed tests passed; 3 ignored), and `git diff --check`. Cargo ran outside the Windows filesystem sandbox because path canonicalization is blocked there. No literal colours found outside `theme.rs` in production UI source.
- Limitations: toggle/navigation messages were driven through production update logic; native mouse clicks, hover tooltips, higher DPI, Linux/macOS and physical readers were not verified. Library/content/details/device actions remain placeholders for later items; save-error toasts remain №11. No commit, PR or remote CI run was created. №7 was not started.

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

done

### Post-implementation notes

- 12 October 2026: implemented Unicode-lowercase author/title sorting, stable BookId index, standalone selection (plain/Ctrl/Cmd/Shift, keyboard movement/ranges/select-all/clear), modifier tracking and focus-loss reset. Rescans retain existing selected IDs and drop removed IDs; changing libraries clears selection. Home renders a virtual 36 px list with three-row overscan, spacers, responsive columns, format tags, real font-metric ellipsis and a text cache bounded to 2048 entries. Selection opens the details placeholder.
- Added the item 7 preview to the HTML mockup first; the user explicitly approved empty-library actions, header refresh and warning list in this chat before production implementation. Home now includes those states and list/grid controls. Grid is a placeholder, and list is usable by default until item 9 adds grid/default/persistence. Full metadata/Convert remain item 13.
- Ran the real iced renderer probe on Windows at scale 1 with 20,000 synthetic books: catalog build 56.875 ms; 209 warm scroll redraw intervals, mean 5.544 ms, p95 10.429 ms, maximum 11.675 ms. These are redraw event intervals, not GPU presentation time or a physical mouse-wheel test. Captured and inspected list at 1100×700 and selected rows/warnings/empty/grid placeholder at 760×520. Details, reproduction, captures and raw output: [docs/UI_CATALOG.md](docs/UI_CATALOG.md).
- Local checks passed: scripts/check-ui.ps1 (workspace fmt, locked build, all-target UI Clippy/tests: 40 library tests plus 1 spike test), default-workspace locked all-target Clippy and tests (107 passed, 3 ignored), git diff --check. Windows sandbox blocked initial fixture canonicalization/atomic writes; final Cargo checks ran outside it. Initial Clippy findings were fixed before the final pass.
- Limitations: sorting is Unicode lowercase code-point order, not locale collation. Physical mouse/keyboard input, higher DPI, Linux/macOS and physical readers were not verified. Renderer data is synthetic and preferences isolated. No commit, PR or remote CI run was created. №8 was not started.

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

done

### Post-implementation notes

- 12 October 2026: added expandable Library/Formats/Authors navigation, counts from scan snapshots (one count per book/format), Unicode-lowercase author grouping, active facet states, filtered Home headings/counts and unfinished coordinator job count. Library is expanded by default; expansion persists for the session only. Facet selection retains stable selected IDs; clicks, Shift ranges and Home keyboard actions use the currently visible order.
- Search now has an automatically focused input, empty/no-result/loading states, result count and the shared virtual list; it always hides details and ignores Home grid mode. Unicode-lowercase title/author substring matching uses a precomputed immutable index on a background worker. One outstanding worker coalesces intermediate edits; generation checks reject old results/errors across A→B→A, cleared queries, rescans and library changes. Leaving Search preserves query/results. Diacritics remain significant.
- Executed the native Windows probe with RealServices/coordinator/core scanner on a six-book on-disk fixture; every scanned format and normalized author path returned the expected visible order. Library file bytes stayed unchanged. Inspected native Home at 1100×700 and facet/Search screens at 760×520, including input focus, Cyrillic/Polish queries and restoration after Jobs/Settings. This is a fixture directory scan with a metadata.db marker, not a personal Calibre database or physical reader verification.
- Native debug search measurement: 20,000 synthetic catalog entries, nine edits at a 30 ms typing interval, asynchronous SearchChanged→SearchFinished mean 11.836 ms, maximum 17.624 ms. This includes scheduling/delivery, not GPU presentation or physical keystroke latency. Captures, raw output, architecture, reproduction and limitations: [docs/UI_NAVIGATION.md](docs/UI_NAVIGATION.md).
- Passed scripts/check-ui.ps1 (workspace fmt, locked UI build, all-target UI Clippy/tests: 45 library tests plus one spike test), default-workspace locked all-target Clippy and tests (107 passed, 3 ignored), git diff --check. Cargo checks ran outside the Windows filesystem sandbox because it restricts canonicalization and atomic fixture writes.
- Limitations: no accent folding or locale collation; expanded author sidebar is not virtualized (20,000-book probe used one author); physical mouse/keyboard, higher DPI, Linux/macOS, remote CI and physical readers were not verified. Grid/covers, full Jobs, Settings and metadata/Convert remain their later items. No commit or PR was created. №9 was not started.

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

done

### Post-implementation notes

- 12 October 2026: implemented background cover discovery/decoding, 192×264 thumbnails, atomic disk PNGs under the OS cache directory and keys including library/source path, BookId, mtime, size and pipeline version. Decoder dimensions/allocation are limited. A 300-entry LRU includes missing/failed covers; deterministic title-colour placeholders and once-per-book error logs replace failed images without toasts. Cache-write failure still displays the source thumbnail.
- Home/facets now default to a responsive virtual grid with fixed-height rows and spacers. Four workers load only visible cards plus one forward viewport. Generation checks ignore stale results; outstanding workers retain slots through scrolling/rescans/library changes. Grid/list choice persists via the existing serialized atomic ui.toml writer; Search remains list-only. Ctrl/Cmd and Shift selection share the list Selection. List thumbnails now use the same cache; it is available for the later details-pane item.
- Executed separate native Windows cold/warm processes with 5,000 catalog IDs backed by six real JPEG covers read-only. This is a repeated-cover load fixture, not 5,000 different Calibre books or physical reader verification. Final cold traversal: 222.093 s, 280.2 MiB peak working set, 16 ms heartbeat mean 16.003 ms / p95 17.314 ms / max 76.319 ms. Warm: 29.503 s, 281.6 MiB peak, heartbeat mean 15.998 ms / p95 29.701 ms / max 37.816 ms. Both kept at most 300 ready handles/four workers. Warm reused all 5,000 PNGs without timestamp changes. Initial cold diagnostics included screenshot encoding; corrected sampling excludes startup/captures and PNG encoding runs on a worker. These intervals measure scheduled UI-message delivery, not physical input or GPU frame time.
- Inspected native grid at 1100×700 and list thumbnails at 760×520. Architecture, reproduction, screenshots, raw measurements and limitations: [docs/UI_COVERS.md](docs/UI_COVERS.md). Tests cover metadata/source invalidation, disk reuse/corruption/write failure, oversized/missing/invalid images, LRU, placeholder determinism, grid geometry, fast-scroll/hidden stale replies, worker limits and view saves.
- Checks passed: scripts/check-ui.ps1 (workspace fmt, locked UI build, all-target Clippy/tests: 51 library tests plus one spike test), default-workspace locked all-target Clippy and tests (107 passed, three ignored), git diff --check. Cargo checks ran outside the Windows filesystem sandbox because it blocks canonicalization, Clippy lookup and atomic fixture writes. No commit, PR or remote CI run was created.
- Limitations: physical mouse/keyboard, higher DPI, Linux/macOS and 5,000 different source images were not verified. Disk variants are retained; memory alone has an eviction cap. Grid keyboard navigation, larger details images and tint remain their later items. №10 was not started.

---

## Item №10. Device bar

### What and why

The bottom bar is always visible and represents the currently connected reader. The core already provides `DeviceWatcher`/`DeviceTracker` with events for connect, disconnect, replace, path change and polling failure ([docs/DEVICE_WATCH.md](docs/DEVICE_WATCH.md)). The UI must only render those states correctly.

### How

- Start `DeviceWatcher::start(interval, publish)` at launch; `publish` forwards `DeviceEvent` into the iced runtime through a subscription backed by a channel (`iced::stream::channel`), returning `false` from `publish` when the receiver is gone so the watcher stops itself. Stop the watcher on exit (it joins its thread).
- Hold the latest snapshot in `State`: devices list, selected id, warnings. Use `DeviceTracker::select` semantics: a single reader is selected automatically, several readers are not.
- Bar states from the mockup: *none* (grey dot, "No reader connected", hint/manual-folder action), *one* (green dot, name, mount point, free space, Send button), *several* (amber dot, "N readers found · choose one ▾" in the reader-name position until selected; then the selected name with ▾, mount point, free space and Send). The reader-name dropdown replaces the separate "Choose reader…" button, as requested by the user. `PollFailed` and detection warnings appear as a small warning text with the diagnostic (full text in the toast area, №11).
- Free space: query available bytes for the mount point. The core does not expose free space today (checked: no such function in `volumes`/`transfer`). Add `volumes::free_space(mount_point) -> Result<u64, Error>` to the core (Win32 `GetDiskFreeSpaceExW`, `statvfs` on Unix) with tests on temporary directories and mock failures. Refresh at each poll; a failure shows "free space unknown", not an error.
- Reader-name dropdown (G4): when several readers are present, clicking the name/▾ opens a list upward from the left side of the bar. Entries show name, label, mount point, filesystem and read-only flag; the active reader has a checkmark. "Select folder manually…" remains in the dropdown and is directly accessible when none/one is detected. It uses the core manual-selection API ([docs/DEVICE_DETECTION.md](docs/DEVICE_DETECTION.md)) with the same validation. Read-only readers are shown but Send is disabled with an explanation.
- The Send button itself is wired in №11; here it is rendered and enabled/disabled by state.
- Tests with the fake device feed: connect, disconnect, replace, two readers, selection, poll failure, read-only device, watcher shutdown.

**Completion criterion:** plugging a folder fixture through the core's test snapshot support produces all bar states in the right order; no thread remains after exit.

### Status

done

### Post-implementation notes

- 12 October 2026: G4 was revised and signed off by the user. Multiple readers use a dropdown at the reader name with a fixed-size SVG chevron and checked active entry; there is no separate chooser button. The list opens upward on the left. A single reader's name is plain text; manual selection stays accessible through a folder icon or the disconnected bar. The user views the HTML mockup in their own browser; do not start a web server to render it again.
- Implemented `services/devices.rs`: core DeviceWatcher worker, iced channel subscription, complete snapshots, session selection, detection diagnostics, manual-root revalidation on every poll, and per-poll free-space refresh even when DeviceTracker suppresses unchanged events. The worker uses the core's injectable `start_with` API to include manual readers and space queries. Receiver closure returns false from publish; subscription cancellation and service shutdown stop/join the worker. Poll failure clears selection; recovery with several readers requires a new choice.
- Added the permanent bar and picker with name/path/free space, selection checkmark, Kobo/Kindle type selector, native Browse, core manual validation, disabled read-only/empty-selection Send and warning tooltips. Manual request generations reject replies after dismissal/exit; feed revisions reject polls started before a new manual choice. Long paths alone are ellipsized so free-space value/unknown status remains visible. Send is only rendered/enabled here; job submission and toast diagnostics remain №11.
- Added `volumes::free_space(&Path) -> Result<u64, Error>` using GetDiskFreeSpaceExW on Windows and statvfs on Unix, without write probes. Three core tests cover a temporary Unicode directory with unchanged library bytes, invalid paths, injected OS failures and zero/large results. Six new UI tests cover the core-classified folder feed, connect/disconnect/replace/path changes, multiple readers, warnings/failure/recovery, read-only, space refresh/stale paths, manual validation/stale replies, closed receiver and iced-stream cancellation. Shutdown assertions verify join and no subsequent polls.
- Executed native `devices_probe` on Windows at 760×520 with two on-disk folder fixtures classified from injected core volume snapshots. Captured ten states and inspected dropdown/manual/warning/read-only/unknown-space layouts; chevron and free-space status are visible. This is fixture verification, not a physical or mounted-reader test. Reproduction, captures and limitations: [docs/UI_DEVICES.md](docs/UI_DEVICES.md).
- Checks passed: default-workspace locked all-target Clippy with warnings denied and locked tests (110 passed, three ignored), scripts/check-ui.ps1 (workspace formatting, locked UI build, all-target Clippy/tests: 57 library tests plus one spike test), and git diff --check. Cargo checks ran outside the Windows filesystem sandbox due to its canonicalization restrictions. No commit, PR or remote CI was created. Linux/macOS execution, higher DPI, physical mouse/keyboard/folder-dialog use and physical readers were not verified. Core names reader kinds (Kobo/Kindle), not individual hardware models. №11 was not started.

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

done

### Post-implementation notes

- 12 October 2026: resolved G2 in the HTML reference and received user sign-off before Rust implementation. Added a bottom-right notification stack above the permanent reader bar: success/cancellation notices expire after five seconds, errors stay until closed. The reference now matches the native ✓/! card icons and wrapping stage text. Existing UI notices are also visible and dismissible.
- Wired Send through Services and the coordinator with one request per selected catalog book, preserving selection chronology and capturing the reader and stored KEPUB preference. Empty selection, absent/read-only reader, Setup/loading and shutdown block sending. Capacity/destination conflicts wait in FIFO order. Progress/queue count and Cancel appear in the bottom bar; a selected reader only shows its own transfers. Preparation shows stage text, copying shows actual bytes/percentage. Jobs now renders Running, Up next and the last 20 completions, per-job Cancel, Clear finished and an empty state; sidebar badge counts unfinished jobs. Core errors are mapped to path-aware conflict/permission/partial-transfer messages; cancellation remains Cancelled.
- Added per-executor injectable mounted-volume enumeration in core and a matching RealServices constructor. Production still uses OS enumeration, and injected feeds pass the same reader recognition, identity/read-only, destination, copy/publication and cleanup checks. Three real-core UI integration tests cover three sends and destination-byte verification, unchanged complete library snapshots, repeated-send conflicts without overwrite, disappearance after a copy chunk with an actual leftover .part, and cancellation of one waiting and one copying job with cleanup. Five new library tests cover selection/request order/preferences/guards, FIFO cancellation, error categories, progress stages, badges, history and toast expiry.
- Checks passed locally on Windows: default-workspace locked all-target Clippy with warnings denied and locked tests (110 passed, three ignored); scripts/check-ui.ps1 (workspace formatting, locked UI build, all-target Clippy/tests: 62 library tests, three send integration tests and one spike test); embedded mockup JavaScript syntax compilation; git diff --check. Cargo checks ran outside the Windows filesystem sandbox because it denies workspace canonicalization.
- Ran native jobs_probe and inspected six 760×520 captures for preparation, copying, Running/Up next, cancellation, success and sticky error. Fixed minimum-width stage wrapping and path-row constraints so the reader name, free-space status and actions remain visible. These are scripted layout states; real fixture transfers were checked separately in tests. Reproduction, screenshots and limitations: [docs/UI_JOBS.md](docs/UI_JOBS.md).
- No physical or mounted-reader transfer, Linux/macOS execution, higher DPI, physical input or multiple simultaneous destinations were manually verified. Synthetic ebook bytes require no external conversion tool. Core sends report Preparing throughout any preparation conversion, then Copying; no distinct conversion substage is exposed. No commit, PR or remote CI run was created. №12 was not started.

---

## Item №12. Settings screen

### What and why

Several later items read user preferences; the screen that edits them must exist first. It also resolves the export folder (G3) that conversion needs.

### How

- Settings screen rows (as in the mockup): Cover tint (toggle), Prefer KEPUB for Kobo (toggle), Library folder (path + Change…), and the new Export folder (path + Change…, G3). Window blur was removed after №2. Update the mockup with the Export folder row first.
- Changing the library folder reuses the Setup validation, saves the config, clears selection and cover caches for the old library, and starts a scan. A failed validation keeps the old library.
- Export folder: chosen with the folder picker; must exist, be writable, and be outside the library. The core already enforces this in `converter::validate_output_dir`, which is `pub(crate)`; make it public (or add a thin public wrapper) so the Settings screen shows the same verdict the conversion API will give, instead of re-implementing it. Store in `ui.toml`.
- Every change is saved immediately through the atomic writer; a failed save shows a toast and keeps the previous value in the file.
- The Cover tint toggle is stored now; its effect is implemented in №14. Until then it has no visible effect and the notes must say so. Window blur is excluded after №2.
- Tests: each setting round-trips; invalid export folder (inside the library, missing, read-only); library change while a scan is running; failed save keeps the old file.

**Completion criterion:** all settings persist across restarts; changing the library works end to end with a second test library.

### Status

done

### Post-implementation notes

- 12 October 2026: resolved G3 in the interactive HTML reference and received user sign-off before Rust implementation. Settings now renders Cover tint, Prefer KEPUB for Kobo, Library folder and Export folder, with wrapping paths and native folder pickers. The first-convert folder prompt is designed in the reference; its conversion wiring belongs to №13.
- Preference edits use the existing serialized/coalesced atomic ui.toml writer. Save failures show a sticky toast and restore the last saved values. Cover tint is stored only and has no visible effect until №14; the stored KEPUB preference already applies to sends. No window blur setting was added.
- Settings library changes reuse Setup's Calibre validation and atomic config writer with the native path preserved. Cancellation, invalid libraries and failed saves retain the old library and scan. Successful saves clear selection/metadata/catalog and the in-memory cover cache, cancel the old scan, switch to Home and scan the new library; obsolete results are ignored. Existing disk thumbnails remain safe to reuse because keys include the source path.
- Made converter::validate_output_dir public and added write-access checking shared by Settings and conversion: canonicalization, existing directory, outside the active and other Calibre libraries, not read-only, then an exclusively created empty probe that is removed. The library-protection checks run before any probe write. Export validation is asynchronous and rejects stale replies after a library change.
- Added update tests for toggle/export coalescing and rollback, cancelled/failed library pickers, invalid library choices during a scan, library switching and obsolete export results. Added core validation coverage (inside-library, missing, file, read-only and other-library destinations, no residual probe, unchanged library snapshot) and a Windows read-only ui.toml replacement test.
- Added tests/settings_flow.rs, executing real iced task streams, RealServices, config/settings IO and coordinator scans against two temporary fixture libraries. Folder-picker responses are injected; widget focus/scroll effects are skipped. Verified toggles/export persistence across a fresh services/state restart, failed export/library validation, real failed settings replacement on Windows, library switching during an active scan, nonempty selection/cover-cache clearing, the second catalog and unchanged complete snapshots of both libraries. These are test libraries, not personal libraries or physical readers.
- Checks passed locally on Windows: scripts/check-ui.ps1 (workspace formatting, locked UI build, all-target Clippy with warnings denied and tests: 66 library tests, three send tests, one Settings integration test and one spike test); default-workspace locked all-target Clippy with warnings denied and locked tests (111 passed, three ignored); git diff --check. Cargo checks ran outside the Windows filesystem sandbox after a sandboxed attempt exposed its path-canonicalization access restriction.
- Ran native settings_probe and visually inspected three 760×520 renderer captures: defaults, saved toggles with a long Unicode export path, and a sticky error. The probe uses isolated config and scripted layout states; native folder-picker interaction is not automated. Reproduction, screenshots and limitations: [docs/UI_SETTINGS.md](docs/UI_SETTINGS.md).
- No physical reader, personal library, Linux/macOS execution, higher DPI or physical input was verified. Folder access can change after validation and is rechecked by conversion. No commit, PR or remote CI run was created. №13 was not started.

---

## Item №13. Details pane, metadata, multi-select, Convert

### What and why

The details pane is the right-hand pane in the design: single-book metadata, a "N selected" summary for several books, and the Convert action for all selected books. It needs asynchronous metadata loading with stale-response protection, because the user can change selection faster than OPF files are read.

### How

- Visibility: the pane is shown on Home/Library screens when the toggle is on; any selection change forces it open (as designed). The toggle persists in `ui.toml`. The pane is never shown on Search, Jobs, Settings.
- Single selection: cover (from the №9 cache, larger size), title, then fields Author, Publisher, Year (from `Metadata.date`), Language, Series (+ index), Formats, Rating, identifiers, tags, description. Hide empty fields. Metadata is requested with `JobRequest::Metadata { book }` or a direct `read_metadata` call in a blocking task; key the response with the №4 guard (request generation + `BookId`), so A→B→A does not show stale data. While loading show the title/author from the catalog immediately (the same fallback as `Metadata::with_fallback`), then fill in.
- Metadata errors: the book still shows with fallback fields; the error appears as a muted line "Metadata could not be read" with the cause available in a tooltip/toast, not a blocking error.
- Description text: the core already returns it normalised (paragraphs preserved); render with wrapping and let it scroll with the other metadata in the pane. After user review, the separate description scrollbar/height cap was removed to avoid nested scrolling.
- Multi-selection: "N selected" counter, first five titles and "+ M more", Convert for all.
- Convert menu: request backends once at start (`JobRequest::Backends`) and refresh on demand; compute targets with `converter::target_formats(files, backends)` for the selected book(s). For several books show only targets available for **all** of them and list the skipped books. Routes without a tool show "no route" with the reason (missing, broken, no KEPUB support) from `Backend` diagnostics. Never show a target the core would reject.
- Convert action: for each selected book submit a conversion job (`JobRequest::ConvertFiles` with `library`, `output_dir` = the export folder from №12, `files`, `target`). If no export folder is set, ask once (G3) and store it. Conflicts with an existing output name are reported, never overwritten. Result toast says "Converted · saved to <folder>" with an Open folder action. The converted file is **not** added to the library, and the UI says so only where needed (Jobs row text).
- Conversion progress shows stage only (core does not provide a reliable percentage).
- Tests: stale metadata rejection (A→B→A), fallback on metadata error, multi-selection targets intersection, "no route" reasons, conversion job creation per book, missing export folder flow, name conflict.

**Completion criterion:** converting a fixture book with the fake backend produces the correct job and result; if `ebook-convert` is installed on the developer machine, one real TXT → EPUB conversion is run and recorded (otherwise recorded as not verified).

### Status

done

### Post-implementation notes

- 12 October 2026: implemented the details pane with a larger cached cover, immediate catalog fallback, asynchronous OPF fields, empty-field omission, wrapped/scrollable description and a muted metadata-error line with cause tooltip. Selection-triggered requests use generation + BookId guards; A → B → A and rescan responses cannot replace current metadata. Existing persisted visibility and Home/Library-only layout rules remain in effect.
- Multi-selection shows the count, first five titles and remaining count. Backend discovery runs at startup and can be refreshed. Convert enables only the intersection of core target_formats for all selected books. Disabled targets expose per-book reasons, including already available formats, missing/broken tools and no KEPUB support.
- Convert captures the original books/library/target and submits one ConvertFiles job per book through the coordinator. Implemented the approved G3 export dialog: shared asynchronous validation, Cancel/Browse/Use folder, atomic preference persistence before submission. Invalid or cancelled choices, failed saves and library changes do not start the pending batch. Conversion has stage-only progress; completion offers Open folder and Jobs notes that outputs are outside the library. No output is added to the catalog.
- Real conversion exposed that existing output names were reported as InvalidInput. Changed both existing and racing publication conflicts to ErrorKind::Conflict, retaining no-overwrite behavior, with core fake-runner coverage. The UI now gives the existing-file/not-overwritten diagnostic.
- Added five update tests for selection metadata/fallback/guards, targets and diagnostics, multi-book job requests, first-export persistence/cancellation/failure and synthetic conversion results/conflicts/Open folder. Added an opt-in real-services integration test using iced task streams, injected folder-picker replies and installed ebook-convert. Ran it successfully with Calibre 8.16.2: synthetic TXT → EPUB (final run 27,085 bytes), ZIP/EPUB MIME checks, export preference persistence, completion action, unchanged complete test-library snapshot; a repeat conversion reported a conflict and preserved both the EPUB and library bytes. This is a temporary test library, not a personal library or physical reader.
- Checks passed locally on Windows: scripts/check-ui.ps1 (workspace formatting, locked UI build, all-target Clippy with warnings denied, 71 library tests, three send integration tests, one Settings integration test and one spike test; the external-converter test is ignored by default and was run separately); default-workspace locked all-target Clippy with warnings denied and locked tests (111 passed, three ignored); git diff --check. Final example edits were formatted and checked with UI all-target Clippy. Cargo ran outside the Windows filesystem sandbox after path canonicalization was denied inside it.
- Ran native details_probe and visually inspected five 760×520 captures: cached synthetic cover/single selection, scrolled metadata and description, eight selected books with the Convert menu, muted metadata error/missing converter and the Unicode export dialog. Reproduction, screenshots and limitations: [docs/UI_DETAILS.md](docs/UI_DETAILS.md).
- Linux/macOS execution, higher DPI, physical input, native folder-picker interaction, Open folder launching, personal libraries and physical readers were not manually verified. Cover tint remains №14. No commit, PR or remote CI run was created. №14 was not started.

- Post-review amendment, 12 October 2026: user approved adjustable text/cover sizes, a draggable persisted details divider and contrasting Convert routes. Updated the mockup first, then implemented the changes on the existing screens. Added two library tests for appearance bounds and drag/coalescing/save rollback, extended real Settings restart coverage and adapted virtualization/ellipsis to the chosen sizes. Final scripts/check-ui.ps1 passed (formatting, locked build, all-target Clippy/tests; shared-tree UI library: 74 passed and one parallel Kindle smoke test ignored). The Settings restart integration, three send integration tests and spike test passed. Default-workspace locked Clippy/tests and git diff --check passed. Fixed a watcher-fixture race exposed during verification; concurrent Kindle production changes were left to the other agent. Renderer captures and limits are in [docs/UI_APPEARANCE.md](docs/UI_APPEARANCE.md). No work on №14 was started.

- Additional user correction, 12 October 2026: removed the separate description scroll container and height cap; the complete wrapped description now scrolls with the metadata through the single content scrollbar. Updated the HTML reference and docs. scripts/check-ui.ps1, default-workspace locked Clippy/tests, mockup JavaScript syntax and git diff --check passed.

---

## Item №14. Visual effects: window opacity and blur

### What and why

The design calls for a translucent window over a blurred desktop, as in Sonora. The №2 spike rejected it because wgpu's default Windows surface is opaque. That conclusion was too broad: presenting through a DirectComposition swap chain makes it possible. This item delivers it as Settings controls: an opacity slider and a blur toggle. (An earlier version of this item tinted the details pane with a colour from the cover; the user rejected it. The effect must be transparent, not cover-coloured.)

### How

- Windows: select DX12 with `WGPU_DX12_PRESENTATION_SYSTEM=visual` before iced starts, create the window transparent without a redirection bitmap, present with premultiplied alpha. This needs two small patches in `vendor/iced_wgpu` and `vendor/iced_winit` (inactive unless that variable is set); documented in `vendor/README.md`.
- Settings: **Window opacity** (10–100 %, step 5, default 70; 100 % is opaque) sets the fill alpha live. **Blur window** (toggle, default on) draws a translucent window over a blurred desktop (acrylic via `SetWindowCompositionAttribute`, which, unlike the DWM system backdrop, keeps blurring when the window is unfocused); off gives a translucent window over a sharp desktop. Fills stay opaque until the effect is really in place, and use a darker glass palette while translucent.
- On Windows with this mode the toolbar is the title bar (own drag, maximise, minimise, close and invisible resize handles); the OS caption was a visibly lighter separate strip.
- If the platform or OS refuses, stay opaque, show one toast, keep the stored values and disable both controls with an explanation. Other platforms are unchanged.
- Update the HTML reference (opacity slider and blur toggle over the wallpaper gradient).
- Do not implement per-element backdrop blur (decision above) or any cover-colour effect.
- Tests: alpha mapping, apply/clear/refusal state machine, settings defaults/clamp/round-trip.
- Manual check on Windows 11 over a high-contrast backdrop at several opacities, focused and unfocused; record Windows 10, macOS and Linux as verified or not verified.

**Completion criterion:** moving the slider in Settings changes the window from opaque to a translucent window over a blurred desktop on the developer's Windows machine, and the blur stays when the window loses focus, with screenshots in the notes.

### Status

done

### Post-implementation notes

- 12 October 2026: first implemented a cover-colour wash (details pane, then whole window). The user rejected it: the effect must be transparent, not picked from the cover. That code (cover tint service, tint cache, wash layer, tests, probe) was removed.
- Investigated the №2 no-go. wgpu 27 supports `Dx12SwapchainKind::DxgiFromVisual` (premultiplied alpha), but iced_wgpu ignores `WGPU_DX12_PRESENTATION_SYSTEM` and prefers straight alpha, which DirectComposition rejects (`Invalid surface`). A first run also produced a flat grey window: a dark 35 % fill over an opaque white redirection surface. Fixed with two marked patches in `vendor/iced_wgpu` and `vendor/iced_winit` (wired via `[patch.crates-io]`, directory excluded from the workspace). Details: [docs/UI_BLUR.md](docs/UI_BLUR.md), [vendor/README.md](vendor/README.md); the spike document carries a pointer.
- Implemented `blur.rs` (environment setup, alpha mapping, acrylic apply/clear, `BlurState`), `Message::SetWindowOpacity`/`SetWindowBlur`/`BlurApplied`, `sync_blur` after every update, transparent frame clear via `app::style`, translucent root fill and details panel, and the two Settings controls (disabled with a reason when unavailable). The Cover tint toggle and `cover_tint` are replaced by `window_opacity: u8` (default 70) and `window_blur: bool` (default on); an old `cover_tint` key in ui.toml is ignored. `window-vibrancy` and `libloading` are Windows-only normal dependencies; one small `unsafe` block calls `SetWindowCompositionAttribute`.
- Follow-up the same day, after side-by-side comparison with Sonora, in three rounds. (1) The OS caption was a separate, lighter strip: on Windows with blur available the toolbar is now the title bar (own drag/maximise/minimise/close and invisible resize handles); D3 is revised for that configuration. (2) The window looked grey and milky: the acrylic layer lifts any backdrop (about 52/255 over black), so translucent mode now uses a darker glass palette. (3) Sonora's own control turned out to be an *opacity* slider plus a separate blur toggle, where 15 % means very transparent; I had first modelled a "blur strength" slider and misread dark-desktop measurements, then briefly inverted it. The controls now match Sonora's. Legacy blur-behind (`apply_blur`) was tried and rejected: it does not blur under a DirectComposition visual.
- **Focus loss:** the first acrylic implementation (`window-vibrancy::apply_acrylic`, which on Windows 11 22H2+ sets the DWM system backdrop) turned solid when the window lost focus. Applying acrylic through `SetWindowCompositionAttribute` keeps the blur. Measured with a scripted focus hand-off over a striped backdrop: system backdrop mean chroma 26 → 70, accent policy 11.6 → 8.8 (the control run confirms the harness sees the difference). Capture: [opacity-15-unfocused](docs/ui-item-14/opacity-15-unfocused.png). The capture script was first written so that it exposed the user's own desktop in a screenshot; that file was deleted immediately (never committed or shared) and the script now keeps a backdrop window on top.
- Scrollbars (user report, same day): they were 10 px thick, always visible, and in the grid view sat in the middle of the centre pane because the scrollable shrank to the card width. Now a shared style (`theme::vbar`, `theme::scroll(state)`): 4 px rounded overlay scroller; visible while the wheel/keyboard/drag scrolls, on hover or drag of the bar, and for 5 idle seconds afterwards, then transparent (the 100 ms timer runs only while visible); the grid scrollable is full width so the bar sits at the pane edge. The scroller length is proportional to content/viewport (so it is long only for short lists); it was not capped. Test `scrollbars_show_on_activity_and_hide_after_five_idle_seconds`; real captures checked visible/hidden; physical mouse wheel not exercised.
- Updated the HTML reference (opacity slider, blur toggle, wallpaper gradient behind a translucent app); its JavaScript compiles. Added tests for alpha mapping and the translucency/backdrop state machine plus updated settings tests, `scripts/capture-blur.ps1` and the `blur_probe` example. Real OS-level desktop captures at 15, 40, 70 and 100 % and unfocused at 15 % are in [docs/ui-item-14](docs/ui-item-14/); inspected.
- Checks passed locally on Windows: scripts/check-ui.ps1 (workspace formatting, locked UI build, all-target Clippy with warnings denied, tests); default-workspace locked all-target Clippy with warnings denied and tests; mockup JavaScript syntax compilation; git diff --check. No commit, PR or remote CI run was created.
- Not verified: Windows 10; real mouse use of the custom title bar (drag, double-click maximise, buttons, edge resizing: wired to iced window operations and compiled, not exercised with input); Windows 11 snap layouts on the maximise button (lost); flicker while resizing; battery/power behaviour; other GPUs/drivers; 150 %/200 % scaling; macOS (vibrancy not wired) and Linux (opaque). The vendored patches must be re-applied on an iced upgrade. №15 was not started.

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
