# Send, Jobs and notifications (UI item 11)

Date: 12 October 2026. G2 notification design approved by the user before Rust implementation.

Send takes the selected books from the catalog in selection chronology (Ctrl-click appends, range expansion adds new books in visible order, Select all uses visible order). Each request captures the book path, author, selected reader and the persisted `prefer_kepub` preference. Empty selection, missing/read-only reader, Setup, library loading and shutdown block submission. The coordinator preserves FIFO requests when core capacity or destination reservations prevent submission; changing readers does not redirect queued requests. Progress in the bar belongs to the selected reader; when no reader remains selected it can still show the outstanding transfer with Cancel until its result arrives.

`widgets/jobs.rs` renders Running, Up next and Done, with cancellation, Clear finished and an empty state. UI and coordinator retain the last 20 completions in completion order. Results use actual copy-byte progress, clamped to 0–100%; preparing/starting/converting stages have text and no percentage. Core send currently reports Preparing throughout file preparation, including any conversion, then Copying. It does not distinguish the conversion substage of preparation; this UI does not invent one.

`widgets/toasts.rs` renders a scrollable bottom-right stack above the permanent bar. Success and cancellation notices close after five seconds (measured from the next 100 ms clock event); errors remain until explicitly closed. Clock timestamps arrive as messages, so update-level lifetime tests do not sleep. The clock subscription stays alive for expiring notifications after the last job completes. Existing settings/picker/metadata notices also become visible as dismissible error cards. The HTML reference was aligned with the implemented ✓/! icons and wrapping stage text.

Core errors are translated before their categories are discarded at the UI boundary: conflict explains that the existing file was not overwritten, permission denial includes the path, and PartialTransfer keeps the possible leftover `.part` path and reconnect/check guidance. Cancelled results remain Cancelled in Jobs. Success uses the captured reader name, not the current selection. Core currently names reader kinds (Kobo/Kindle), rather than hardware models.

## Validation on Windows

- Default workspace: `cargo clippy --locked --all-targets -- -D warnings` and `cargo test --locked` passed (110 tests, three existing ignored tests).
- `scripts/check-ui.ps1`: workspace formatting, locked UI build, all-target Clippy with warnings denied, and all-target tests passed (62 UI library tests, three new integration tests, one existing spike test).
- `git diff --check` and embedded mockup JavaScript syntax compilation with Node passed.
- Cargo ran outside the restricted Windows filesystem sandbox, which denies canonicalization of workspace paths.

Reproduce fixture IO with:

```powershell
cargo test -p rulibre-ui --locked --test send_jobs
```

The three tests drive app messages through RealServices, the real coordinator, JobExecutor, reader recognition and transfer IO. They reuse the core's synthetic library/snapshot support. JobExecutor's per-instance `with_volume_source` injects mounted-volume enumeration only; production `new` retains native enumeration, and the same reader identity, read-only, path, publication and cleanup checks still run before/during copying.

1. Send three directly compatible fixture books, observe complete copy-byte results, compare destination bytes with their sources, and compare every library filename/byte including metadata.db/OPF before and after. Repeat the sends: three conflicts, no destination overwrite and unchanged library.
2. Remove the injected volume after the first 64 KiB copy chunk: a Failed job and sticky toast contain the actual leftover `.part` path. The partial file remains in the fixture and the library snapshot is unchanged.
3. Hold volume revalidation after a first copy chunk, cancel one waiting and one running job, then release the worker: both are Cancelled, the waiting book was never written, the partial copy was cleaned, and reader/library file snapshots match the originals.

Fake-service/coordinator tests additionally cover request identity/order/preferences, queue retries, waiting/running cancellation, progress/stage mapping, error categories, badge counts, bounded history and notification expiry/dismissal.

## Native layout evidence

```powershell
cargo run -p rulibre-ui --example jobs_probe --locked
```

This example creates isolated temporary configuration and a core-classified folder reader, renders scripted job states in a real iced window, saves six screenshots at 760×520 and exits. Its screenshots are layout evidence for synthetic states; they are not screenshots of the real integration transfers above. No user configuration or personal library is used.

Inspected all six captures: [Preparing](ui-item-11/preparing-760.png), [Copying](ui-item-11/copying-760.png), [Running and Up next](ui-item-11/jobs-760.png), [Cancelled](ui-item-11/cancelled-760.png), [Success](ui-item-11/success-760.png), [Sticky error](ui-item-11/error-760.png). Stage/queue text wraps at minimum width; paths shorten with a full-path tooltip and free-space status remains visible. Explicit path-row widths prevent the reader name from clipping when progress disappears.

## Limits

No physical reader or mounted-reader transfer was tested. The injected volume disappearance is a deterministic fixture failure, not an OS unmount. Fixture ebook bytes are synthetic; this item does not verify external conversion tools. Linux/macOS execution, higher DPI, large-window layout, physical clicks/keyboard cancellation and multiple simultaneous destination transfers were not manually verified. The local HTML reference was syntax-checked, not browser-interaction tested; no web server was started. No commit, PR or remote CI run was created. Settings editing remains item 12; the default stored preference is already used for sending.
