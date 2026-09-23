# Security Policy

*(Eine deutsche Kurzfassung findest du am Ende dieser Datei.)*

## Reporting a vulnerability

Please **do not** report security vulnerabilities through public GitHub issues, discussions, or
pull requests. Please report privately by email instead:

**security@jcs-net.de**

A PGP key for this address is published via [Web Key Directory](https://wiki.gnupg.org/WKD)
(WKD) and should be auto-discovered by most modern mail/GPG clients. Fingerprint:

```
C166 608E BFA2 BD80 7D93  5471 4A36 D2FF E207 B4ED
```

Please include, as far as you can:

- A description of the vulnerability and its potential impact
- Steps to reproduce (kodi-spot-scan version, Windows version, Kodi version)
- Any proof-of-concept configuration or request that triggers the issue

You should receive an acknowledgement within a few days. This is a small, solo-maintained
open-source project without a dedicated security team, so please allow reasonable time for a fix
before any public disclosure. I'll coordinate a disclosure timeline with you once the report is
confirmed.

## Supported versions

Only the latest released version is supported with security fixes. Please make sure you're on
the current version before reporting, and update to the fixed version as soon as a patch is
released.

## Scope

kodi-spot-scan reads filenames and modification times from configured local network shares
(SMB/CIFS), and sends JSON-RPC requests over HTTP to a Kodi instance you configure. It does not
read the contents of media files, and does not store or transmit Kodi credentials — the current
design assumes network-trusted, unauthenticated (or already OS-level authenticated) access to
both the shares and the Kodi instance. If your setup requires authenticated access to either,
please open an issue first so this can be designed in deliberately rather than bolted on.

Settings are stored in a local, unencrypted text file under `%APPDATA%\kodi-spot-scan\` — treat
that folder like any other local application settings file. This is not designed to be exposed
to, or run on behalf of, an untrusted or multi-tenant environment.

Reports particularly welcome around:

- Path handling between the Windows share view and the `smb://` paths sent to Kodi (e.g.
  path/directory traversal via crafted folder names)
- Anything that would let a JSON-RPC request be sent to a host other than the one configured
- Handling of untrusted or malformed content in `.nfo` filenames

General Kodi, tinyMediaManager, or Windows/network security issues are out of scope here — please
report those to the respective project.

---

## Auf Deutsch (Kurzfassung)

**Sicherheitslücken bitte nicht** als öffentliches Issue melden, sondern per E-Mail an
**security@jcs-net.de**. Ein PGP-Key für diese Adresse ist über
[Web Key Directory](https://wiki.gnupg.org/WKD) (WKD) auffindbar, Fingerprint
`C166 608E BFA2 BD80 7D93 5471 4A36 D2FF E207 B4ED`.

Unterstützt wird nur die jeweils aktuelle Version. Da es sich um ein Solo-Projekt ohne
dediziertes Security-Team handelt, bitte etwas Zeit für einen Fix einplanen, bevor öffentlich
darüber gesprochen wird.

kodi-spot-scan liest nur Dateinamen/Änderungszeiten von den konfigurierten Netzwerkfreigaben und
sendet JSON-RPC-Anfragen an eine konfigurierte Kodi-Instanz - keine Zugangsdaten werden
gespeichert oder übertragen, das aktuelle Design setzt vertrauenswürdigen, unauthentifizierten
Zugriff auf Freigaben und Kodi voraus. Einstellungen liegen unverschlüsselt unter
`%APPDATA%\kodi-spot-scan\`.
