# kodi-spot-scan

Targeted Kodi library updates, triggered the moment
[tinyMediaManager](https://www.tinymediamanager.org/) finishes writing a movie's or TV show's
`.nfo` file — instead of waiting on a slow, occasionally unreliable full library rescan.

> **Status:** early development, no releases yet. This README will grow (installation,
> configuration, usage) as the first working version comes together — see
> [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the current design if you want the details
> now.

## The problem this solves

Kodi can keep its video library in a shared MariaDB/MySQL database, so several Kodi instances on
different devices can all see the same library. But triggering Kodi to actually pick up a newly
added movie or show normally means running a full library rescan of the whole share — slow, and
on some devices unreliable enough to fail outright.

`kodi-spot-scan` watches the relevant SMB/CIFS shares for `.nfo` files written or updated by
tinyMediaManager, and — once you confirm it's ready — tells a specific Kodi instance (via its
JSON-RPC API) to scan *just that one folder*, using Kodi's `VideoLibrary.Scan`. No more full
rescans for a single new movie.

## How it works, in short

1. `kodi-spot-scan` polls the configured shares for new or changed `.nfo` files (polling rather
   than filesystem change notifications, because those aren't reliable over SMB/CIFS).
2. Detected folders show up in a live list in the app.
3. You review the list and explicitly trigger a send — nothing is scanned automatically or
   silently.
4. The Windows path is translated to the corresponding Kodi source path (`\\server\share\...` →
   `smb://server/share/...`) and sent as a `VideoLibrary.Scan` request to a Kodi instance you
   configure — independent of which machine `kodi-spot-scan` itself runs on.

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the full design and the reasoning behind
these choices.

## Requirements

- Windows 7 or later
- A Kodi instance reachable over the network with remote control via HTTP enabled (*Settings →
  Services → Control*)
- tinyMediaManager (v3) preparing `.nfo` files on the watched shares

## Links

- Report a bug or request a feature: [issue tracker](https://github.com/chrisschm/kodi-spot-scan/issues)
- [Security policy](SECURITY.md)
- [Contributing guide](CONTRIBUTING.md)
- [Code of Conduct](CODE_OF_CONDUCT.md)
- [Changelog](CHANGELOG.md)

## License

GPL-3.0-or-later - see [LICENSE](LICENSE).

This program is free software: you can redistribute it and/or modify it under the terms of
the GNU General Public License as published by the Free Software Foundation, either version 3
of the License, or (at your option) any later version.

---

## Auf Deutsch (Kurzfassung)

**kodi-spot-scan** beobachtet die SMB/CIFS-Freigaben, auf denen tinyMediaManager `.nfo`-Dateien
für Filme/Serien ablegt, und löst - nach deiner ausdrücklichen Bestätigung - bei einer
konfigurierten Kodi-Instanz per JSON-RPC (`VideoLibrary.Scan`) einen gezielten Scan **nur des
betroffenen Ordners** aus, statt eines langsamen und mitunter unzuverlässigen kompletten
Bibliotheks-Scans.

**Status:** frühe Entwicklungsphase, noch keine Releases. Details zur Architektur stehen bereits
in [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

**Voraussetzungen:** Windows 7 oder neuer, eine per HTTP fernsteuerbare Kodi-Instanz im Netzwerk, sowie
tinyMediaManager (v3), das die `.nfo`-Dateien auf den beobachteten Freigaben pflegt.

**Weitere Links:** [Fehler melden](https://github.com/chrisschm/kodi-spot-scan/issues),
[Sicherheitsrichtlinie](SECURITY.md), [Mitwirken](CONTRIBUTING.md),
[Verhaltenskodex](CODE_OF_CONDUCT.md), [Changelog](CHANGELOG.md).

**Lizenz:** GPL-3.0-or-later. Dieses Programm ist freie Software: Sie können es unter den
Bedingungen der GNU General Public License, wie von der Free Software Foundation
veröffentlicht, weitergeben und/oder modifizieren, entweder gemäß Version 3 der Lizenz
oder (nach Ihrer Option) jeder späteren Version.
