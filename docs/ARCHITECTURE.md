# Architecture

This document explains how `kodi-spot-scan` is built and, more importantly, *why* — the
reasoning behind each non-obvious decision, so future changes can respect it instead of
accidentally undoing it.

## The problem, restated

Kodi can share one video library (backed by a MariaDB/MySQL database) across several installed
Kodi instances on different devices. Getting a newly added movie or TV show into that library
normally means running a full library rescan of the whole share from *some* Kodi instance — slow,
and on some devices unreliable enough to fail outright, especially over Wi-Fi on a laptop.

Since the library already gets essentially all of its metadata from tinyMediaManager-authored
`.nfo` files and artwork on disk (rather than Kodi's own scrapers), there's no need for a *full*
rescan when a single movie or show is added or edited — a targeted scan of just that one folder
(Kodi's `VideoLibrary.Scan` JSON-RPC method, given a `directory` parameter) is enough, and is
fast and reliable by comparison.

`kodi-spot-scan` exists to make triggering that targeted scan easy: watch the shares tMM writes
to, and let the user fire off a scan of exactly the folder that changed.

## Module breakdown

twinBASIC's Project Explorer shows standard modules and class modules with the same icon
(only forms are visually distinct), so object names carry a classic VB6-style prefix:
`mod` for a standard module, `cls` for a class module, `frm` for a form.

| Object | Kind | Responsibility |
|---|---|---|
| `modAppConfig` | Standard module | Loads/saves settings (watched shares, poll interval, Kodi target host/port) from the INI file under `%APPDATA%\kodi-spot-scan\`. No per-instance state, so a module rather than a class. |
| `modPathTranslator` | Standard module | Pure conversion between the Windows UNC path the watcher sees (`\\server\share\...`) and the `smb://` path Kodi's JSON-RPC API expects (`smb://server/share/...`). Kept isolated because it's the piece most likely to need a fix for an edge case (special characters in a title) without touching anything else. Stateless, so a module. |
| `clsShareWatcher` | Class module | Periodically polls the configured shares for new/changed `.nfo` files, tracks (path, last-modified) state on disk so a restart doesn't re-flag everything, and raises an event per detected folder. Needs a class module for the event and the per-instance state. |
| `clsKodiClient` | Class module | Wraps the JSON-RPC HTTP call to Kodi (`VideoLibrary.Scan`, plus a connectivity check) against a configured host/port. |
| `frmMain` | Form | Live list of detected folders (path, detected-at, status) with multi-select and an explicit "send to Kodi" action, plus a short status/history log. |
| `frmSettings` | Form | Edits watched shares, poll interval, and the Kodi target; includes a "test connection" action. |

## Key design decisions

### Polling instead of filesystem change notifications

`ReadDirectoryChangesW`-style change notifications require the SMB server to support and
correctly forward "change notify" requests. This isn't reliable enough across the range of
NAS/server implementations and across reconnects to build the watcher on — a notification stream
can silently stop working without any visible error. Polling (periodically walking the watched
trees and comparing `.nfo` modification times against the last known state) is less elegant but
doesn't have this failure mode: worst case, a change is noticed one poll interval later, never
missed outright.

### `.nfo` files are the only trigger

tinyMediaManager writes artwork references into the `.nfo` itself, so every relevant change (a
new movie, updated metadata, new artwork) shows up as a change to that one file. Watching
anything else (image files, subtitles, etc.) would only add noise without adding signal.

### No automatic or silent scanning

Detected changes only ever populate a list for the user to review; a scan is only ever sent to
Kodi as the direct result of an explicit user action. There is deliberately no "stable for N
seconds → send automatically" heuristic and no persisted approval queue to work through later —
targeted scans are cheap enough that immediate, explicit action beats either automation or
batching.

### Path translation is a simple, deterministic string rule

For this maintainer's setup, the SMB server name and share names are identical between how
Windows sees them (`\\ds\video\...`) and how Kodi's source is configured (`smb://ds/video/...`),
so translation is a straight prefix/slash substitution — no lookup table, no credentials needed
in the request (Kodi resolves the matching source's own stored credentials, if any, from the
path alone). This is an assumption specific to a "flat" SMB setup without embedded credentials
in the source URI; a setup with `smb://user:pass@host/...` sources, or server/share names that
differ from their Windows view, would need `PathTranslator` extended with an actual mapping
rather than a blind substitution.

Encoding of special characters (accents, `&`, `#`, etc.) in folder/file names between the two
path forms hasn't been verified empirically yet — treat this as open until tested against a few
real "difficult" titles.

### The Kodi target is independent of which machine runs the watcher

`kodi-spot-scan` may run on a different physical machine than the Kodi instance it should
trigger (e.g. watching from a laptop, but scanning via a more reliable desktop Kodi instance).
Kodi's JSON-RPC-over-HTTP interface already supports this natively (any host/port reachable on
the LAN), so the Kodi target is just a configuration value — never assumed to be `localhost`.

### Settings location: `%APPDATA%\kodi-spot-scan\` (Roaming), plain files

Chosen over the Windows registry (`SaveSetting`/`GetSetting`) for transparency (a config file can
just be read, copied, or attached to a bug report) and over `%ProgramData%` because settings here
are per-user, not shared across multiple Windows accounts on one machine, and because the tool
isn't expected to need an installer that could set up `%ProgramData%` ACLs in advance. Roaming
rather than Local because these are small user preferences, not machine-tied cache data — even
though, in this maintainer's non-domain setup, Roaming vs. Local makes no practical difference.

The same folder holds the watcher's persisted "last seen" state and a short send history/log,
so everything the tool keeps lives in one place that's easy to inspect or reset.

### No credential handling

The current setup assumes trusted, unauthenticated (or already OS-level authenticated) access to
both the watched shares and the target Kodi instance's JSON-RPC endpoint — `kodi-spot-scan` never
stores or transmits a password. If a future setup needs authenticated access to either, that
should be designed in deliberately (see `SECURITY.md`) rather than added ad hoc.

## Known limitation: no CI build

twinBASIC does not currently have a command-line compiler
([twinbasic/twinbasic#508](https://github.com/twinbasic/twinbasic/issues/508) is open and
unresolved), so there is no automated build/release pipeline yet — builds are done manually in
the twinBASIC IDE. This should be revisited once that issue is resolved upstream.

## Version control and the twinBASIC project

The `.twinproj` project container is a binary format and isn't meaningfully diffable. twinBASIC
can export a project to UTF-8 text files instead
(`twinBASIC_win32.exe export <twinproj_path> <export_folder_path> --overwrite`), which is what
gets committed under `src/`; note that `export` always creates a `<export_folder>/<ProjectName>/`
subfolder rather than exporting directly into the given folder — see `export.bat`, which
automates the export and flattens that back out. `export.bat` is gitignored, since it hardcodes
paths specific to the machine it runs on (the twinBASIC install location, the `.twinproj`
path); copy `export.bat.example` to `export.bat` and fill in those paths for your own setup.

Only `src/Sources/`, `src/Settings`, and `src/Resources/` are the project's own content and need
to be committed. `src/Packages/`, `src/ImportedTypeLibraries/`, `src/References/`, and
`src/Miscellaneous/` hold copies of referenced compiler packages (VB/VBA/VBRUN compatibility
layers, WinNativeCommonCtls, etc.) that `export` may bundle in — these are gitignored, since
twinBASIC resolves referenced packages from its own central package store (keyed by the GUIDs
recorded in `Settings`) on import, not from what's in this folder. Confirmed by testing an
`import` (twinBASIC's `File > New/Open Project > Import from folder...`, or the `import`
CLI command) against a `src/` tree with those folders empty: the project loads with all
referenced packages resolved, no missing-package prompt.

The binary `.twinproj` itself, and the `Build/` output folder, stay local/gitignored — only the
exported `src/` tree is version-controlled.

### Getting changes back into the live project

`export`/`export.bat` is one-way: dev project (`.twinproj`) → repo (`src/`). Changes made
the other way — for example, source files under `src/` edited directly (in an editor, or by a
Claude session working against the repo checkout) rather than through the twinBASIC IDE — need
to be brought back into the `.twinproj` before the IDE will see them; otherwise the next
`export.bat` run regenerates `src/` from the (unchanged) `.twinproj` and silently discards them.

This direction goes through the IDE, not a script, and takes two steps: close the project if
it's open, then use **File > New/Open Project... > Import from folder...** (the same dialog
used for creating a project from an exported tree — see above), pointed at the repo's `src/`
folder. That dialog only takes a source folder, not a target path — it opens the imported
content as a project in the IDE without saving it anywhere. To bring it back into the existing
`.twinproj`, follow up with **File > Save As...** and overwrite that file's path.

Confirmed (2026-09-23): changed `Form1`'s `Caption` in `src/Sources/Form1.tbform` outside the
IDE, then imported `src/` and saved over the existing `.twinproj` this way — the changed
caption showed up on reopening. No CLI command or script is needed for this direction; there
is no `import.bat`.
