# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

Initial development. Nothing released yet.

### Added

- `modPathTranslator`: UNC <-> `smb://` folder path conversion.
- `modAppConfig`: settings in `%APPDATA%\kodi-spot-scan\kodi-spot-scan.ini` (UTF-16LE).
- `clsKodiClient`: JSON-RPC over HTTP (`JSONRPC.Ping`, `VideoLibrary.Scan` with `directory`).
- `clsShareWatcher`: polling `.nfo` watcher with persisted state and silent first-run baseline.
- `modScanWorker`: the shares are walked in a background thread, so the window stays responsive.
- `modStrings` and `Resources/STRING/Strings.json`: localized UI (English as fallback, German).
- `frmMain`: detected-folder list, explicit "send to Kodi", log, native Win32 status bar.
- `frmSettings`: shares, poll interval, Kodi target with connection test.
- `modFolderPicker` and a "..." button in `frmSettings`: pick a share folder in a dialog
  (modern file dialog in folder mode, classic folder dialog as fallback); folders on mapped
  network drives are converted to their UNC path.
- `frmMain` remembers its size, position and maximized state (`[Window]` in the INI).
- Own application icon.
