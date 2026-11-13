# Changelog

All notable public changes are recorded here. The project follows a practical
semantic-versioning scheme for the desktop application.

## [Unreleased]

## [0.4.0] - 2026-11-12

### Added

- Four-layer long-term memory: transcripts, notes, wiki pages, and a bounded
  `MEMORY.md` injection index.
- Session/project/global scope isolation, deterministic local embeddings, optional
  OpenAI-compatible remote embeddings, hybrid semantic retrieval, feedback, pinning,
  archival, and Dreaming maintenance.
- Memory center UI and APIs for search, editing, wiki reading, transcript inspection,
  synchronization, backfill, and Dreaming runs.
- Persistent Router/Worker supervision with heartbeat, stall recovery, bounded retries,
  desktop notifications, and restart recovery.
- General multi-agent collaboration DAGs with handoff context, reviewer/arbiter roles,
  failure isolation, and one final report per collaboration run.
- Windows Job Object/AppContainer sandbox integration and file watcher triggers.
- Tauri 2 desktop packaging with NSIS and MSI outputs.

### Changed

- Task folders are selected explicitly and retained per task with an allowed-root
  boundary.
- Model settings can be saved and tested from the desktop UI.
- Desktop windows hide to the tray while supervision continues.

### Security

- Added path boundaries, sensitive-file filtering, approval gates, optional sandbox
  fallback, and expanded public-repository ignore rules.

## Earlier versions

Earlier local iterations were not published as a stable source release. Their build
outputs and data remain outside this distribution.