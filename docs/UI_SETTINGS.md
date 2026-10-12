# Settings screen (item №12)

Date: 12 October 2026.

The Settings screen has Window blur, Prefer KEPUB for Kobo, Library folder and
Export folder rows. It retains the sidebar and permanent reader bar; the details
pane is hidden. Long paths wrap, and the screen scrolls if necessary.

Preferences are written immediately to `ui.toml` with the existing atomic
writer. Outstanding writes are serialized and rapid edits are coalesced. A
failed write restores the last saved values and shows a sticky error toast.
Window opacity and Blur window are described in [UI_BLUR.md](UI_BLUR.md) (item №14).
The stored KEPUB preference already controls subsequent sends from item №11.

Library Change opens the native folder picker. `RealServices::save_library`
reuses Setup's core Calibre validation and atomic config writer, retaining the
native path throughout. Cancellation, validation failure and save failure keep
the old library and scan. Only a successful save clears selection, metadata,
catalog and in-memory cover cache, cancels the old scan, and starts a new scan
on Home. Old scan and cover responses cannot replace the new catalog. Disk
thumbnails remain reusable and are isolated by absolute source path in their
cache keys.

Export Change opens the native folder picker and validates off the UI thread
with the now-public `converter::validate_output_dir`. This same core function is
used by conversion. It rejects missing paths, files, paths inside the active or
any other Calibre library, and read-only directories. Canonicalization resolves
aliases before protection checks. An exclusive temporary empty file tests write
access outside the library and is removed before returning. Validation does not
create the selected directory. Stale replies after a library change are ignored.
As with conversion, permission or mount changes after validation can still make
a later operation fail.

The G3 HTML design was approved before Rust implementation. It also previews
the first-convert folder prompt; wiring Convert and that prompt belongs to item
№13. Settings currently uses the native folder picker directly.

## Verification

`tests/settings_flow.rs` executes real iced task streams, RealServices, atomic
file IO and the coordinator against two temporary Calibre fixtures. Only folder
picker responses are injected; widget focus/scroll operations have no window in
this test. It checks both toggles and export persistence, a fresh services/state
restart, failed folder validation, a read-only export folder, a real failed
replacement of a read-only `ui.toml` on Windows, library switching during an
active scan, selection/cache clearing and the second catalog. Complete snapshots
of both libraries stay unchanged; no probe files remain in the export folder.
The fixture database is a marker, not a real SQLite database.

Update-level tests also cover cancelled/failed pickers, preference write
coalescing, save rollback, old export replies and obsolete scan results. Core
tests cover output-directory validation and preserved library snapshots.

`cargo run -p rulibre-ui --example settings_probe --locked` creates an isolated
native window, saves preferences through the real update/services path and
captures these states at 760×520:

- [Default settings](ui-item-12/settings-default-760.png)
- [Saved preferences and long path](ui-item-12/settings-saved-760.png)
- [Sticky export error](ui-item-12/settings-error-760.png)

The three images were visually inspected on Windows. The error is a scripted
layout state; actual validation failures are exercised separately in tests.
The probe does not automate physical input or the operating system picker.

## Limitations

No physical reader, personal library, manual native-picker interaction, Linux or
macOS execution, or higher-DPI verification was performed for this item. The
Cover tint toggle was later replaced by the Window blur slider (item №14). Export folder selection does not perform any
conversion; that flow remains item №13. No commit, PR or remote CI run was created.
