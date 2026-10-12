# Cover discovery

`rulibre_core::cover::find_cover(book_dir)` lazily discovers a Calibre cover.
It returns `Result<Option<CoverFile>, Error>`; `CoverFile` contains a canonical
`path`, modification time (`SystemTime`) and byte length (`u64`). These metadata
values let clients invalidate their image caches without changing `Book` or
`ScanReport`.

Only the exact filename `cover.jpg`, matched ASCII case-insensitively, is
considered. Discovery accepts regular files and follows links whose canonical
targets stay inside the canonical book directory. Directories and targets
outside that directory are ignored. Broken links and directory, resolution or
metadata access failures return an error with path context. When multiple case
variants exist, candidates are checked in native path order; the first accepted
regular file wins. An existing directory with no accepted cover returns
`Ok(None)`; a missing book directory returns an error.

The operation reads directory entries and filesystem metadata only. It never
opens, decodes or writes the image, and never changes the Calibre library.
Unreadable image contents can still have accessible metadata: discovery then
succeeds, and a later UI decode may fail. Metadata access denial is returned
as an error, not treated as a missing cover. Image decoding, resizing and cache
storage belong to the client; no image dependency is added to the core.

The scanner currently follows links without a containment check. Cover discovery
uses the explicit, stricter containment requirement of UI plan item №3. Like
other path prechecks, it does not protect against concurrent filesystem changes
between discovery and a later client read.

Tests use synthetic temporary fixtures, including a 512 MiB file, metadata access
failure injection, and a library byte snapshot. Unix tests also cover internal,
external and broken links, plus a file with no content-read permissions. Windows
tests cover discovery while another handle prevents content reads. Unix-specific
tests have not been executed on the Windows developer machine.
