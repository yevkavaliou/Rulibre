# Item 8: sidebar facets and Search

12 October 2026. Windows, native iced renderer, debug build, scale 1.

The sidebar now navigates Home, Search, Jobs and Settings. Library starts
expanded, with independently expandable Formats and Authors. Expansion survives
navigation during the session; it is not written to ui.toml. Formats count books,
not files: duplicate files in the same format count once. Authors group by the
same Unicode lowercase normalization used for catalog sorting, preserving the
first catalog spelling for display. Counts and the immutable search index are
built with the catalog at scan completion. Jobs shows the number of waiting and
running coordinator jobs. Long author names are truncated and have a tooltip.

Facet navigation uses `Screen::Home { filter }` and the same list/grid control as
Home. Headers show `Library · EPUB` or the author's display spelling and the
filtered count. Filtering preserves selected IDs, including hidden books. Mouse
and keyboard selection receive the filtered visible order; an anchor outside
that order falls back to a single selection. Rescans refresh the current filter
and remove IDs that disappeared from the whole library.

Search focuses its input when opened, renders the virtual list even when Home
is in grid mode, and hides the details pane. Empty/whitespace queries have no
results. Matching is a Unicode lowercase substring within either title or author.
Diacritics remain significant: `ŻÓŁW` matches `Żółw`, but `zolw` does not.
Leaving Search preserves the query and results, including across Jobs/Settings.

Search uses an Arc of pre-normalized strings and runs with spawn_blocking. At
most one search worker is outstanding; intermediate edits coalesce into the
latest query. A generation guard rejects obsolete results and errors, including
A → B → A. Query edits, rescans and library changes invalidate results. A scan
completion schedules the retained query against the new snapshot. Clearing a
query immediately clears results, and late worker replies cannot repopulate them.

## Verification

Run `cargo run -p rulibre-ui --example navigation_probe --locked` from the
repository root. The probe creates six book directories with EPUB/PDF fixture
files, a metadata.db setup marker and isolated preferences. Production
RealServices/coordinator/core scanner load those books from disk. Every format
and normalized author path is exercised through production navigation, and
matching visible orders are asserted. The fixture marker is not a Calibre SQLite
database; this scanner reads the directory tree. Library bytes were checked
unchanged after the run. This is a disk fixture, not a personal library or a
physical reader test.

Inspected native captures: expanded facets and counts at 1100×700, filtered
format/author lists at 760×520, focused blank Search, Cyrillic/Polish results and
the 20,000-result virtual list at minimum width. Search retained its query and
results after visiting Jobs and Settings. Actions and typing were dispatched as
production messages, not physical mouse/keyboard input.

- [Home and facets](ui-item-8/home.png)
- [Format filter](ui-item-8/format.png)
- [Author filter](ui-item-8/author.png)
- [Focused empty Search](ui-item-8/search-empty.png)
- [Cyrillic search](ui-item-8/search-cyrillic.png)
- [Polish search](ui-item-8/search-polish.png)
- [Restored Search](ui-item-8/search-restored.png)
- [20,000-result list](ui-item-8/search-20k.png)
- [Raw native probe output](ui-item-8/probe.txt)

The performance phase uses 20,000 synthetic catalog entries and nine query
changes with a 30 ms typing interval. Time from dispatching SearchChanged to
receiving SearchFinished includes background scheduling and result delivery:

| Measurement | Result |
|---|---:|
| Mean search round trip | 11.836 ms |
| Maximum search round trip | 17.624 ms |
| Samples | 9 |

These are asynchronous update round trips, not GPU presentation time or physical
keystroke latency. Expensive substring scans do not execute on the UI thread;
the list builds only viewport rows. Authors in the sidebar are not virtualized,
so an expanded list with thousands of distinct authors still needs a performance
pass in item 15. Search's 20,000-book measurement uses one synthetic author.

Checks passed: scripts/check-ui.ps1 (workspace fmt, locked UI build, all-target
UI Clippy and tests: 45 library tests plus one spike test), default-workspace
locked all-target Clippy and locked tests (107 passed, 3 ignored), git diff --check.
New tests cover facet counts/deduplication/case grouping, Unicode search,
20,000-book search budget, filtered selection and navigation, coalescing/stale
responses/empty query/restoration, rescans and hidden Search selection.
Cargo checks ran outside the Windows filesystem sandbox because it restricts
fixture canonicalization and atomic writes.

Higher DPI, Linux/macOS, physical input/readers and remote CI were not verified.
Grid/covers remain item 9, the full Jobs screen item 11, Settings item 12 and
metadata/Convert item 13. No commit or PR was created. Item 9 was not started.
