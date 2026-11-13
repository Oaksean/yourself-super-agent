# Contributing

Thanks for helping improve Local Super Agent.

## Before you start

- Use Windows 10/11 x64 for the desktop and sandbox integration.
- Use Node.js 24 or newer.
- Use the Rust `stable-x86_64-pc-windows-gnullvm` toolchain pinned by
  `src-tauri/rust-toolchain.toml`.
- Install LLVM-MinGW UCRT, for example:
  `winget install MartinStorsjo.LLVM-MinGW.UCRT`.
- Do not commit API keys, databases, logs, generated installers, private test data,
  screenshots containing personal information, or files under `runtime/`,
  `native/`, `data/`, or `logs/`.

## Development

```powershell
node bin/super-agent.mjs
```

Open <http://127.0.0.1:4317>. Configure an OpenAI-compatible endpoint in Settings
or set `DEEPSEEK_API_KEY` in the current shell. Never put a real key in a tracked
file.

Build the desktop shell:

```powershell
.\scripts\build-tauri.ps1
.\scripts\build-tauri.ps1 -Build
```

## Tests and checks

Run the complete test suite and syntax checks before opening a pull request:

```powershell
.\scripts\build-sandbox.ps1
$files = (Get-ChildItem tests -Filter '*.test.mjs').FullName
node --test --test-reporter=spec $files
npm run check
```

`build-sandbox.ps1` compiles the public C# host into the ignored `native/` directory.
Without that prerequisite, the sandbox-host tests are expected to fail on a clean
checkout.

Some tests use a temporary workspace and stub model calls. Tests must not require a
real API key or public network access. If a Windows sandbox provisioning test is
environment-dependent, state that clearly in the pull request rather than weakening
the assertion silently.

## Pull requests

- Keep changes focused and describe user-visible behavior.
- Add or update tests for routing, persistence, sandboxing, memory, or collaboration
  changes.
- Preserve these boundaries: Router dispatches and summarizes; Workers perform
  filesystem/command work; writes and commands remain approval-gated by default.
- Do not weaken path validation, secret filtering, process isolation, or the
  single-final-report semantics of collaboration runs.
- Update documentation and `CHANGELOG.md` when behavior or configuration changes.
- By contributing, you agree that your contribution is licensed under the
  repository's MIT License.

## Commit hygiene

Do not rewrite published history to hide a leaked secret. Revoke the credential,
rotate it, and follow the security report process. Maintainers may reject a change
that cannot be safely published.