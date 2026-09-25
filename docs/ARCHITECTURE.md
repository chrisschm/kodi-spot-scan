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
| `modScanWorker` | Standard module | The file system walk of `clsShareWatcher`, run in a background thread (`CreateThread` + `AddressOf`, hence a standard module). Delivers per share a flat list of `.nfo` paths, timestamps and target folders. See "Background scan thread" below. |
| `clsShareWatcher` | Class module | Periodically polls the configured shares for new/changed `.nfo` files, tracks (path, last-modified) state on disk so a restart doesn't re-flag everything, and raises an event per detected folder. Needs a class module for the event and the per-instance state. |
| `clsKodiClient` | Class module | Wraps the JSON-RPC HTTP call to Kodi (`VideoLibrary.Scan`, plus a connectivity check) against a configured host/port. |
| `modFolderPicker` | Standard module | Folder selection dialog behind the "..." button in `frmSettings` (modern file dialog in folder mode, classic fallback) and conversion of a folder on a mapped network drive to its UNC path (`WNetGetUniversalNameW`). Declares the COM interfaces it needs (`IFileDialog`, `IShellItem`) itself - no type library or package. |
| `modStrings` | Standard module | Access to the localized UI strings (`Res(id, args...)` with `%1`-style placeholders) and the `ResId` enum mirroring `Resources/STRING/Strings.json`. See "Localization" below. |
| `frmMain` | Form | Live list of detected folders (path, detected-at, status) with multi-select and an explicit "send to Kodi" action, plus a short status/history log. Remembers its size, position and maximized state. |
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

The main window's placement is stored in the same INI (section `[Window]`), see "Window
placement" below.

### Watcher details: baseline, TV shows, failure handling

- **Silent baseline.** The first poll of a share without persisted state only records the
  `.nfo` files that exist, without flagging anything - otherwise the whole library would show
  up as "new" on first start (or after adding a share). The state (path -> last-write time) is
  persisted after every poll, so a restart doesn't re-flag anything either.
- **TV shows are reported at show level.** Below a folder containing a `tvshow.nfo`, a
  new/changed episode or season `.nfo` flags the *show* folder, not the season folder: that's
  the level Kodi's TV show scanner works on, and several changed episodes collapse into one
  targeted scan. Movies are reported as the folder containing the `.nfo`.
- **A share is only ever updated from a complete walk.** If a share is unreachable, or any
  folder in it fails to enumerate mid-walk, that share's poll is abandoned and its state left
  untouched. A flaky connection therefore never turns into "everything deleted" followed by
  "everything new" on the next poll.
- **Deletions are not events.** A vanished `.nfo` is just dropped from the state; removing
  items from the library is Kodi's "clean", not a targeted scan.
- **Skipped folders:** reparse points (avoids loops), hidden+system folders, and NAS
  housekeeping folders (names starting with `@` such as Synology's `@eaDir`, `#recycle`,
  `#snapshot`, `$RECYCLE.BIN`).
- **Enumeration** uses `FindFirstFileExW` (basic info level, large fetch), which returns the
  last-write time with the directory listing - no per-file round trip over SMB.
- **No timer in the watcher class.** `frmMain`'s timer starts polls (`clsShareWatcher.Poll`,
  returns immediately) and pumps the watcher every 250 ms (`Pump`), which delivers the results
  once the background walk is done.
- **Detected-but-unsent folders are not persisted** (consistent with "no persisted approval
  queue" above): the watcher's state advances on detection, so a folder that was detected but
  not sent before closing the tool won't reappear by itself.

### Background scan thread

Walking large shares over SMB takes tens of seconds (about 20 s for ~1000 `.nfo` files in
the maintainer's setup), and a single call against an unreachable server can block for the
whole SMB timeout. Done on the UI thread this froze the window ("Not responding"), so the
walk runs in its own thread:

- **Only the I/O moves to the worker** (`modScanWorker`): Win32 API calls, plain strings and
  module-level arrays - no COM objects, no forms, no events, no localized strings, no code
  path that can raise an error. Per share it returns `.nfo` path, timestamp and the folder
  Kodi should scan (show folder below a `tvshow.nfo`), or an error status.
- **Everything else stays on the UI thread** (`clsShareWatcher`): comparison with the
  persisted state, baseline, events, saving. That's in-memory work of milliseconds.
- **Hand-over without shared mutable state:** the UI thread fills the inputs and starts the
  thread; the worker writes its results and finally sets a "finished" flag. The UI thread
  reads results only after seeing that flag, then releases them. Flags are accessed via
  `InterlockedExchange`/`InterlockedCompareExchange` only (kernel32 exports on x86 - fine,
  builds are 32-bit). No subclassing or window messages needed: `frmMain`'s timer polls.
- **Cancel and shutdown:** a cancel flag is checked by the worker per folder and every 256
  directory entries. On close the form cancels and waits up to 3 s; if the worker is stuck in
  an SMB timeout the form closes anyway - the result buffers are never freed while the worker
  may still write to them, and the process end stops the thread.
- Verified before the switch with a stand-alone twinBASIC test (IDE and compiled EXE): two
  workers walking the same share in parallel while the UI thread did string work - identical
  results, no errors.

### File formats in the settings folder

- `kodi-spot-scan.ini` - UTF-16LE with BOM, read/written with the `*PrivateProfileStringW`
  APIs (they only keep non-ASCII characters, e.g. in share paths, in a file that already
  starts with a UTF-16LE BOM).
- `watcher-state.txt` - UTF-16LE with BOM, one tab-separated line per baselined share and per
  known `.nfo`, written to a `.tmp` file first and then swapped in. Deleting it just causes a
  silent re-baseline.

### Localization: string table resource, Windows picks the language

All user-visible text lives in the twinBASIC string table `src/Resources/STRING/Strings.json`
(one entry per string, one `LCID_xxxx` column per language) and is loaded through
`modStrings.Res`. Windows selects the column matching the display language at runtime, so
there's no language setting in the tool itself.

- `LCID_0000` is English and doubles as the neutral fallback for every language without its
  own column; `LCID_0407` is German. Only full LCIDs work - a primary-language-only column
  (`LCID_0007`) was tested and is *not* picked up for a de-DE system.
- JSON has no comments, so entry 1 (`Readme_LanguageColumns`) explains the table to
  translators - in each language's own column.
- Naming: `<Area>_<Element>_<Property>` (e.g. `frmMain_cmdSend_Caption`; areas are the form
  names, `App`, `Msg`, `Err`), mirrored in `modStrings.ResId` with the prefix `rid_`. The JSON
  `name` field has no function yet; it's meant for twinBASIC's planned named access
  (`Resources.Strings.<name>`). Until then the enum is kept in sync by hand.
- ID ranges: 1-99 meta, 100-199 application, 1000-1999 `frmMain`, 2000-2999 `frmSettings`,
  3000-3999 log/status messages, 4000-4999 error messages.
- Captions set in the form designer are placeholders only; every form sets its real captions
  in `Form_Load` (`ApplyCaptions`).

### No ActiveX/OCX dependencies

The status bar in `frmMain` is the native Win32 one (`msctls_statusbar32` from `comctl32`,
part of Windows), created via API, instead of the `MSComctlLib.StatusBar` from
`mscomctl.ocx`: that OCX isn't part of Windows (it would need an admin-installed, registered
copy on every target PC) and only exists as 32-bit. Controls come from twinBASIC's own
packages or the Windows API only.

### Share picker: file dialog in folder mode

The "..." button next to the share path opens the modern file dialog in folder mode
(`IFileOpenDialog` with `FOS_PICKFOLDERS`) rather than the classic folder tree
(`SHBrowseForFolder`): in the tree, a share that isn't mapped to a drive letter is only
reachable through "Network" and network discovery, whereas the file dialog's address bar
accepts `\\server\` directly. The chosen folder is added to the list right away, with the same
checks as a typed path. A folder on a mapped drive (`Z:\Filme`) is converted to its UNC path;
local and `subst` drives are rejected, since only shares can be translated to `smb://`. The
watching PC may well host the share itself, though: such a folder works when entered as a UNC
path (`\\thispc\share\...`), and the rejection message says so. Resolving a local folder to
its share automatically (`NetShareEnum`, longest matching share path) is a possible later
extension. Manual entry stays possible.

### Window placement

`frmMain` saves its restore bounds (`GetWindowPlacement.rcNormalPosition`, pixels), the
maximized state and the system DPI on close and restores them with `SetWindowPlacement` at the
end of `Form_Load`, while the form is still hidden. Restore bounds rather than the current
rectangle, so a window closed maximized reopens maximized *and* restores to its previous size.
A window closed minimized reopens normal (or maximized, if it was before). If it would end up
completely off screen (monitor removed, lower resolution), Windows moves it back into view. If
the display scaling changed in between, the bounds are scaled by the DPI ratio. Requires
`StartUpPosition = Manual` on the form; without saved values the window is centered on the
primary screen.

### No credential handling

The current setup assumes trusted, unauthenticated (or already OS-level authenticated) access to
both the watched shares and the target Kodi instance's JSON-RPC endpoint — `kodi-spot-scan` never
stores or transmits a password. If a future setup needs authenticated access to either, that
should be designed in deliberately (see `SECURITY.md`) rather than added ad hoc.

## Windows version compatibility

Supported (and stated in the README) is **Windows 7 or later**, 32-bit builds only. Beyond
that, the code deliberately stays compatible back to Windows XP wherever that's cheap - this is
a developer-side goal, not a user-facing promise, so it's documented here and in code comments
only, never in the README or user-facing wiki.

- **Feature test with fallback, no version checks.** Newer APIs or flags are tried first; if
  the call fails because the feature doesn't exist, the code falls back to an older
  equivalent. Code comments name the Windows version each variant needs.
- Current cases:
  - `modScanWorker.FindFirst`: `FindFirstFileExW` with `FindExInfoBasic` +
    `FIND_FIRST_EX_LARGE_FETCH` (Windows 7+); on `ERROR_INVALID_PARAMETER` it switches once to
    the standard call (Windows XP/Vista).
  - `modFolderPicker`: `IFileOpenDialog` with `FOS_PICKFOLDERS` (Vista+); if
    `CoCreateInstance(CLSID_FileOpenDialog)` fails (XP: class not registered), falls back to
    `SHBrowseForFolderW` with `BIF_NEWDIALOGSTYLE | BIF_EDITBOX` (XP), where `\\server\share`
    can be typed into the edit box. `SHCreateItemFromParsingName` (Vista+) is only reached
    on the first path.
- **If compatibility can't be kept** for a feature (no reasonable fallback), the minimum
  Windows version is raised *in the build* - the EXE's OS/subsystem version in the PE header,
  so older Windows refuses to start it cleanly - instead of letting it fail at runtime. That
  change and the reason go into this section and the changelog.
- twinBASIC exposes this directly as the project setting **Target OS Version** (sets
  `MajorOperatingSystemVersion`/`MajorSubsystemVersion` etc. in the PE optional header; choices
  from Windows 2000 up to Windows 10). The project currently uses the compiler default,
  Windows XP (v5.1).

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
