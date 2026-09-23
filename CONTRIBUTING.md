# Contributing to kodi-spot-scan

Thank you for considering a contribution! *(Eine deutsche Kurzfassung findest du am Ende dieser
Datei.)*

## Status

This is a young, solo-maintained project, built with [twinBASIC](https://twinbasic.com/). Issues
for design discussion are very welcome even before there's much code to point to.

## Design goals (please keep these in mind for any change)

- **Never scan or act automatically or silently.** Every Kodi scan is triggered by an explicit
  user action. Background watching only ever populates a list for review — it must never itself
  cause a network call to Kodi.
- **Polling, not filesystem change notifications**, for watching the SMB/CIFS shares.
  `ReadDirectoryChangesW`-style notifications aren't reliably delivered over SMB across the
  variety of NAS/server implementations this needs to work with — see
  [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for details. Don't reintroduce a
  notification-based watcher without solving that reliability problem first.
- **`.nfo` files are the only trigger.** tinyMediaManager writes artwork references into the
  `.nfo`, so watching anything else (image files, etc.) would only create noise.
- **The Kodi target is configurable independently of which machine kodi-spot-scan runs on.**
  Don't assume the Kodi instance being scanned is local to the machine running the watcher.
- **Settings live in a plain-text INI under `%APPDATA%\kodi-spot-scan\`** (Roaming), not the
  registry and not `%ProgramData%` — see `docs/ARCHITECTURE.md` for why.

## Before opening a pull request

1. **Target branch:** please branch from and target `main`.
2. **Build verification:** twinBASIC does not currently have a command-line compiler (see
   [twinbasic/twinbasic#508](https://github.com/twinbasic/twinbasic/issues/508)), so there is no
   automated CI build yet. Please make sure the project still builds cleanly in the twinBASIC IDE
   before submitting.
3. **Version control / project export:** the twinBASIC project's source of truth for version
   control is the exported text form under `src/`, not the binary `.twinproj` container directly
   — see the note in `docs/ARCHITECTURE.md`. Please make sure `src/` reflects your latest changes
   (re-export before committing) so diffs stay meaningful.
4. If a change would affect the module boundaries described in
   [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md), please open an issue first to discuss it
   before investing time in a PR.

## Reporting bugs / requesting features

Use the [issue tracker](https://github.com/chrisschm/kodi-spot-scan/issues).

---

## Auf Deutsch (Kurzfassung)

Junges, solo-gepflegtes Projekt in [twinBASIC](https://twinbasic.com/) - Issues zur
Design-Diskussion sind auch ohne viel vorhandenen Code willkommen.

**Design-Vorgaben:** nie automatisch/still scannen (jeder Kodi-Scan braucht eine bewusste
Nutzeraktion), Beobachtung der Freigaben per Polling statt Dateisystem-Benachrichtigungen (wegen
Unzuverlässigkeit über SMB/CIFS), ausschließlich `.nfo`-Dateien als Auslöser, das Kodi-Ziel ist
unabhängig von der ausführenden Maschine konfigurierbar, Einstellungen liegen als Klartext-INI
unter `%APPDATA%\kodi-spot-scan\`.

**Vor einem Pull Request:** von `main` abzweigen und dagegen stellen; da twinBASIC noch keinen
Kommandozeilen-Compiler hat, gibt es noch keine automatisierte CI - bitte vor dem PR manuell in
der twinBASIC-IDE bauen; den Stand unter `src/` (Text-Export) aktuell halten; bei Änderungen an
der Modulaufteilung vorher ein Issue eröffnen.
