# Windows MTP Kindle detection and transfer

The core and iced UI now include present Windows portable devices alongside
mounted filesystem readers. Detection uses SetupAPI's WPD setup class with
`DIGCF_PRESENT`, without PowerShell subprocesses or device writes.

A Kindle requires an Amazon USB vendor ID (`VID_1949`), a friendly name containing
the word `Kindle`, and the Windows MTP service (`WUDFWpdMtp`). Phones, Fire tablets,
and unrelated USB devices are not classified by name alone. The device instance
ID supplies the stable identity; the existing watcher handles snapshots and
disconnects. Enumeration/property failures become discovery warnings without
hiding mounted readers.

MTP descriptors have `filesystem = MTP`, `WindowsPortableDevice` evidence, and
an empty `mount_point`. They have no usable local filesystem path. `is_mtp()`
identifies this transport and `can_transfer()` permits writable MTP devices on
Windows. The iced bar shows USB / MTP and the friendly name in its tooltip,
without filesystem capacity queries or a manual-folder action. Sending uses
Windows Portable Devices COM interfaces on the job worker thread.

The WPD interface path is matched to the exact detected USB instance ID. A
session enumerates storage functional objects and requires exactly one writable
`documents` folder. It does not create missing reader folders. A prepared AZW3 or
MOBI follows the existing Kindle format/conversion policy. MTP jobs reserve the
device identity, without resolving the empty mount path or enumerating volumes.

The upload checks existing names case-insensitively and available storage space
when reported, then creates a WPD data stream. Short writes are handled, progress
reports actual bytes, and cancellation is checked between writes and before
commit. Source size changes fail the transfer. Success requires `IStream::Commit`
and matching uploaded object size. Incomplete transfers use `Revert`; failed
post-commit verification deletes only the newly created object ID. Cleanup
failure reports `PartialTransfer` with both errors and the display destination.
An upload committed without a retrievable object ID reports a cleanup failure
rather than claiming the object was removed.

Returned paths such as `Kindle (MTP)/documents/book.azw3` are display locations,
not local paths. Cancellation cannot interrupt a currently blocked driver call.
MTP does not provide the filesystem transfer's atomic publication contract;
other applications writing the same filename concurrently are outside the
application's device reservation. Reader capacity is still shown as unknown in
the UI; transfer uses WPD capacity if available. Linux/macOS MTP is unsupported.

Verified on 12 October 2026 with a connected Kindle Paperwhite Signature Edition:
both native SetupAPI detection and the UI discovery/model test found the reader.
The real send job transferred a valid generated AZW3, read it back byte-for-byte,
rejected a duplicate without changing the existing content, reverted a cancelled
upload, and removed its own test objects. The synthetic source library was
unchanged. Core and UI library tests and core/UI all-target Clippy passed outside
the Windows filesystem sandbox. Policy tests cover short writes, progress,
cancellation, disconnect errors, changed source size, failed commit, and cleanup
failure. UI tests cover selection, send availability, and disconnect handling.

Reproduce the physical checks with an MTP Kindle connected:

```powershell
cargo test -p rulibre-core native_portable_device_smoke --locked --offline -- --ignored --nocapture
cargo test -p rulibre-ui native_mtp_discovery_reaches_ui --locked --offline -- --ignored --nocapture
cargo test -p rulibre-core native_mtp_documents --locked --offline -- --ignored --nocapture
cargo test -p rulibre-core native_mtp_roundtrip_and_cancel --locked --offline -- --ignored --nocapture
```

The first three checks read the actual device. The roundtrip check requires
`ebook-convert` on PATH and writes a uniquely named synthetic AZW3 to Kindle,
verifies it, and removes only that object; it also tests cancellation cleanup.
A physical unplug during a driver write and visual inspection of the main
window's send flow were not performed. The existing running UI was left intact;
an updated executable is built separately as `rulibre-ui-mtp.exe` while it is open.

API reference: [SetupDiGetClassDevsW](https://learn.microsoft.com/en-us/windows/win32/api/setupapi/nf-setupapi-setupdigetclassdevsw).
Upload reference: [Transferring content to the device](https://learn.microsoft.com/en-us/windows/win32/wpd_sdk/transferring-an-image-or-music-file-to-the-device).
