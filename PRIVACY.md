# Privacy

## Default behavior

Local Super Agent is a local-first desktop application. It runs an HTTP server bound
to `127.0.0.1` by default and does not include telemetry or analytics. Conversations,
tasks, transcripts, memory indexes, approvals, and settings are stored under
`%LOCALAPPDATA%\LocalSuperAgent` for the installed application, or under the configured
development data directory.

## Network destinations

The application contacts a network destination only when needed for configured
features:

- the configured OpenAI-compatible model endpoint for chat, routing, Dreaming, and
  optional remote embeddings;
- search and page-fetch destinations used by the MAS research pipeline;
- GitHub or package mirrors when a build script downloads build tools.

The selected model provider receives prompts, task context, and any context selected
for that request. Review the provider's data-processing terms before using private
or regulated information. Remote embedding is disabled when `embeddingModel` is empty;
the bundled local embedding path runs on the device.

## Workspace access

The user chooses a task folder and grants the application access to that folder.
Workers can read, write, and execute according to the configured capability and
approval gates. The application does not upload the whole workspace automatically.

## Data retention and deletion

Keep backups if you need history. Uninstalling the application is not a reliable way
to delete user data because the desktop installer intentionally preserves
`%LOCALAPPDATA%\LocalSuperAgent` across upgrades and uninstalls.

To erase local user data intentionally, exit the application and remove that directory
after exporting anything you need. This is a manual, destructive action; the project
does not perform it automatically.

## Contact

Use the repository's private security-reporting channel for a privacy or security
sensitive issue. Never attach real credentials or unredacted user data.