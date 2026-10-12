# Cover pipeline and virtual grid (item 9)

12 October 2026. Implementation: `crates/rulibre-ui/src/services/covers.rs`,
`widgets/book_grid.rs`, `widgets/book_list.rs`, and the app's thumbnail messages.

## Behaviour

Home and Library facets default to the cover grid. Grid/list changes use the
existing serialized atomic `ui.toml` writer and roll back on a failed save.
Search always uses the list. Both layouts use the same stable BookId selection;
Ctrl/Cmd toggles and Shift ranges follow the currently filtered order.

Grid cards are 108 px wide, separated by 12 px, with fixed 182 px rows.
`responsive` computes columns from the available width. Only viewport rows plus
one row on either side are built, with spacers for the rest. A Sensor reports
initial viewport size and subsequent resize, including pane toggles. Titles and
authors use the existing font-metric Unicode ellipsis.

The app requests visible covers plus one viewport of forward lookahead. It
starts at most four `Services::load_thumbnail` futures. RealServices discovers
the source through core `cover::find_cover`, decodes and resizes in
`spawn_blocking`, then returns an iced RGBA handle. Source dimensions are limited
to 12,000 px per axis and the decoder allocation limit is 64 MiB. Thumbnails fit
within 192×264 px, approximately twice the card's displayed size. A 300-entry
LRU includes missing/failed outcomes, avoiding repeated IO on every tick.

A changed viewport/order/view increments the request generation. Results from
old generations are ignored. Outstanding workers retain their slots until they
finish, even after rescans or library changes; rapid scrolling cannot multiply
the worker count. Source metadata is checked again after LRU eviction or rescan.
Decode/discovery failures are logged once per book per scan, without toasts.
Missing, loading and failed covers use the same deterministic title-hash colour
and abbreviated title. The list uses this cache for 22×30 px thumbnails.
The reusable cover widget/cache is available for the later details-pane item.

Disk PNGs live under `dirs::cache_dir()/rulibre/covers` (temporary-directory
fallback when unavailable). SHA-256 keys include the BookId, full source path,
modification time, length, thumbnail dimensions and a pipeline version. Full
source paths separate libraries with identical Calibre IDs. PNG writes are
atomic. Corrupt cache files are rebuilt; cache-write failure still displays the
decoded source. Source covers and the library database are never written.

## Verification

Tests cover metadata/path key invalidation, changed source pixels, disk reuse,
corrupt-cache repair, failed-cache-write fallback, missing/invalid/oversized
images, LRU eviction, placeholder determinism, column/viewport geometry,
four-worker limits, fast-scroll stale responses, hidden replies and view saves.

The Windows native probe scans the configured library read-only, then repeats
its six covered books into 5,000 distinct catalog IDs. These are real JPEG cover
files with a synthetic catalog; this is not a 5,000-title Calibre database or a
physical reader check. The traversal advances one viewport after its covers
are ready. It exercises discovery, decoding, disk writes, renderer uploads,
LRU turnover and the production Scrollable operation across all 5,000 IDs.

Results are recorded in `docs/ui-item-9/*.stdout.txt` and `*.memory.txt`.
Cold and warm measurements use debug builds on Windows at scale 1. Warm reuse
checks all 5,000 cache filename/mtime pairs before and after a separate process.
The initial cold run included startup and synchronous screenshot PNG encoding
in its heartbeat sample; retain it as diagnostic evidence only. The final
probe saves screenshots on a worker and samples heartbeat delivery only during
the grid traversal, after first paint and the initial screenshot.

Warm process results: traversal 29.503 s, maximum 300 ready handles and four
workers, peak working set 295,317,504 bytes (281.6 MiB). The 16 ms heartbeat
had 1,753 samples, mean 15.998 ms, p95 29.701 ms and maximum 37.816 ms.
All 5,000 cache timestamp pairs were unchanged; no PNG was rewritten.

Final cold process results: traversal 222.093 s, maximum 300 ready handles and
four workers, peak working set 293,863,424 bytes (280.2 MiB). The 16 ms heartbeat
had 13,705 samples, mean 16.003 ms, p95 17.314 ms and maximum 76.319 ms.
It began with zero PNGs and ended with 5,000. The two processes therefore turn
over the LRU across the entire catalog without retaining all decoded covers.

Local checks passed: `scripts/check-ui.ps1` (workspace formatting, locked UI
build, all-target Clippy and tests: 51 library tests plus one spike test),
default-workspace `cargo clippy --locked --all-targets -- -D warnings` and
`cargo test --locked` (107 passed, three ignored), and `git diff --check`.
Cargo checks ran outside the Windows filesystem sandbox, which blocks
canonicalization, Clippy configuration lookup and atomic fixture writes.

Screenshots inspected: [grid at 1100×700](ui-item-9/grid-1100.png) and
[list at 760×520](ui-item-9/list-760.png).

Reproduction (PowerShell):

```powershell
cargo build -p rulibre-ui --example covers_probe --locked
./target/debug/examples/covers_probe.exe '<library folder>' '<isolated cache folder>'
# Repeat with the same cache folder for a warm process start.
```

Use a cache outside the library. The probe stores temporary preferences and
does not edit the user's saved configuration. Peak process working set was
sampled through `Process.PeakWorkingSet64` every 500 ms; it includes the renderer,
fonts, native runtime, catalog and decoded source buffers, not just the LRU.
Heartbeat intervals measure delivery of scheduled UI messages, not physical
input latency, GPU frame time or presented frames. Warm/cold times include
first paint, cover waits, scripted scrolling and screenshot scheduling.

## Limits

Physical mouse/keyboard responsiveness, higher DPI, Linux/macOS and a library
with 5,000 different source images have not been verified. Disk cache variants
are retained; only the in-memory cache has an eviction limit. Thumbnail images
are suitable for cards/list rows; higher-resolution details/tint work belongs
to the later pane/effects items. Keyboard navigation of the grid belongs to the
keyboard pass; mouse multi-selection already shares the list's Selection.
No commit, PR or remote CI run was created for this item.
