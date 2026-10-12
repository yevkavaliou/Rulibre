# Details and conversion (item 13)

Implemented on 12 October 2026. The right pane is available on Home and Library filters. Selecting a book opens it and persists the toggle. Search, Jobs and Settings hide it.

Single selection immediately renders catalog title/author, then asynchronously reads OPF through the coordinator. Responses carry both request generation and BookId: A → B → A rejects both older results. Missing values use catalog fallback. Read failures leave the book visible with a muted diagnostic and a cause tooltip. Empty metadata fields are omitted; descriptions wrap and scroll with the other metadata in the pane. The separate description scroll area and height cap were removed after user review to avoid nested scrolling. The existing cover cache supplies a 144×198 display from 192×264 thumbnails; missing covers have a placeholder. Cover tint remains item 14.

Multiple selection shows the count, first five titles and remaining count. Converter discovery runs at startup and can be refreshed from the Convert menu. Active targets are the intersection of core `target_formats` for every selected book. Disabled targets show “no route”; tooltips list each unavailable book and its core diagnostic, including existing formats, missing/broken tools and absent KEPUB support. No partial batch silently skips a book.

Convert captures the selected books, active library and target. With no saved export folder, the approved G3 dialog offers Browse, Cancel and Use folder. Validation uses the shared core validator and the batch waits until the atomic settings writer confirms the folder. Cancel, invalid folders, failed settings saves and library changes cannot submit that pending batch. Each book gets a separate `ConvertFiles` job. Progress is stage-only. Completion gives “Converted · saved to …” and Open folder; Jobs explains that the output is outside the library. Existing and racing output names now return the core Conflict category and are never overwritten.

## Verification

- `scripts/check-ui.ps1`: formatting, locked build, all-target Clippy with warnings denied, all-target tests.
- Default workspace: `cargo clippy --locked --all-targets -- -D warnings`, `cargo test --locked`.
- Five new update tests cover selection-triggered A → B → A, metadata-error fallback, common targets and missing/broken/KEPUB diagnostics, captured multi-book jobs, cancelled/invalid/unsaved export choices, synthetic conversion completion/Open folder and sticky conflict errors. Existing guard, selection, settings, coordinator and core fake-runner tests remain in use.
- `cargo test -p rulibre-ui --locked --test conversion_flow -- --ignored --nocapture` executes real services, iced task streams, folder validation/settings IO, coordinator jobs and installed `ebook-convert` (Calibre 8.16.2). It converted a synthetic TXT to EPUB, checked its ZIP signature and EPUB MIME type, unchanged complete library snapshot, saved folder and Open folder action. A second conversion checked a visible name conflict, unchanged EPUB bytes and unchanged library snapshot. The test is opt-in so CI does not require an external converter. Folder-picker replies are injected; no physical reader is used.
- `cargo run -p rulibre-ui --locked --example details_probe` renders isolated scripted states at 760×520. Captures in [ui-item-13](ui-item-13/) cover a synthetic cached cover, full metadata after scrolling, eight selected books/menu, metadata error/missing converter and long Unicode export path. These are native renderer captures, not physical-input automation.

## Limitations

Windows was exercised. Linux/macOS, higher DPI, physical Ctrl/Shift clicks, native folder-picker interaction, Open folder launching, physical readers and personal Calibre libraries were not manually verified. The real conversion test uses a temporary fixture library; the fake completion test writes synthetic bytes and does not invoke a converter. No commit, PR or remote CI run was created. Item 14 was not started.
