# Security Policy

## Supported versions

Security fixes are currently provided for the latest `0.4.x` release.

| Version | Supported |
| --- | --- |
| 0.4.x | Yes |
| < 0.4 | No |

## Reporting a vulnerability

Please do **not** open a public issue for a vulnerability.

After the repository is published, use the host's private vulnerability reporting
channel (for GitHub: **Security > Advisories > Report a vulnerability**). Include:

- affected version and operating system;
- a minimal reproduction or proof of concept;
- impact and required preconditions;
- whether user files, credentials, process isolation, approval gates, or network
  access are involved;
- any suggested mitigation.

Do not include real API keys, personal data, production databases, private
workspace paths, or full logs in a report. Redact secrets and replace private paths
with placeholders.

If private reporting has not yet been enabled on the public repository, contact
the maintainers privately through the repository owner's published security
contact before disclosing details.

## Security model and operator responsibilities

Local Super Agent can read and write files and can execute commands through
separate Workers. The desktop application is not a substitute for operating-system
access control or human review.

Operators should:

- grant the smallest task folder that is sufficient for the work;
- keep workspace-write and command auto-approval disabled unless they understand
  the risk;
- review every command approval and write operation;
- avoid running untrusted prompts, plugins, or generated code with elevated
  privileges;
- keep Windows, WebView2, the application, and the model endpoint up to date;
- protect `%LOCALAPPDATA%\LocalSuperAgent` as sensitive user data.

The Windows Job Object/AppContainer integration reduces the blast radius of Worker
execution but is not a guarantee against every escape, confused-deputy, or
model-driven mistake. Report suspected sandbox escapes privately.

## Secrets

API keys are stored locally and must never be committed. The repository ignores
local configuration, databases, logs, runtime binaries, and common secret files.
A leaked key must be revoked and rotated immediately; deleting the local file is
not sufficient once the key has entered Git history or a release artifact.