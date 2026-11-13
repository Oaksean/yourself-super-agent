# Release Checklist

This repository is the source distribution. Do not commit build outputs, bundled
runtime binaries, databases, logs, installers, or user workspaces.

## Release artifacts

Publish the installers as GitHub Release assets, not as tracked files:

- preferred: `Local Super Agent_<version>_x64-setup.exe` (NSIS);
- optional: `Local Super Agent_<version>_x64_en-US.msi` (MSI);
- always: `SHA256SUMS.txt`;
- optional: source archive generated from the public Git tag;
- optional: an SBOM and third-party license inventory.

The MSI/NSIS files are generated under `src-tauri/target/release/bundle/` and must
remain outside the publishable source tree.

## Pre-release checks

1. Update the version in `package.json`, `src-tauri/Cargo.toml`, and
   `src-tauri/tauri.conf.json`.
2. Update `CHANGELOG.md`.
3. Run the complete test suite and syntax checks from `CONTRIBUTING.md`.
4. Build and smoke-test both fresh install and upgrade paths on a clean Windows VM.
5. Confirm no API key, local database, private path, log, screenshot, `runtime/node.exe`,
   `native/WindowsSandboxHost.exe`, or `src-tauri/target/` is tracked.
6. Generate checksums outside the source directory:

```powershell
Get-FileHash ".\src-tauri\target\release\bundle\nsis\*.exe" -Algorithm SHA256 |
  ForEach-Object { "$($_.Hash.ToLower())  $([IO.Path]::GetFileName($_.Path))" } |
  Set-Content .\SHA256SUMS.txt
```

7. Record the release notes, supported Windows version, required WebView2 version,
   known limitations, and unsigned-binary warning.

## Signing status

The current installers are not code-signed. Windows SmartScreen may warn on first
launch. Do not present an unsigned binary as signed; publish the checksum and the
exact build provenance. Code signing is recommended before a broad public release.

## User data and upgrades

Installed user data lives under `%LOCALAPPDATA%\LocalSuperAgent`. Upgrades and
uninstalls must not silently delete that directory. A clean uninstall should remove
application files while preserving conversations, databases, memory, logs, and
settings unless the user explicitly chooses a data-removal action.

## Source archive

Create a source ZIP from outside the repository directory so it cannot include itself:

```powershell
Compress-Archive -Path '.\open-source\*' -DestinationPath '.\open-source-0.4.0-source.zip' -Force
Get-FileHash '.\open-source-0.4.0-source.zip' -Algorithm SHA256
```

The archive is a convenience snapshot; Git tags are the authoritative source history.