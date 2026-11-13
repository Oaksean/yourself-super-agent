# Third-Party Notices

Local Super Agent is distributed under the MIT License, but it depends on and is
built with third-party software that remains under its own licenses.

| Component | Role | License/source |
| --- | --- | --- |
| Node.js | Bundled JavaScript runtime for desktop builds | Node.js project license; see the Node.js distribution |
| Tauri 2 | Desktop shell and bundler | MIT/Apache-2.0, subject to the upstream project |
| Rust crates | Native desktop dependencies | Versions and source pins are recorded in `src-tauri/Cargo.lock` |
| Microsoft Edge WebView2 | Windows webview runtime | Microsoft terms; the application requires the WebView2 runtime |
| LLVM-MinGW UCRT | Build toolchain/linker | LLVM and MinGW-w64 upstream licenses |
| NSIS | Windows installer builder | zlib/libpng-style NSIS license |
| WiX Toolset | MSI installer builder | Microsoft Reciprocal License / upstream terms |

The repository does not vendor every third-party runtime. A release should identify
the Node.js and WebView2 versions actually shipped or required and retain the
corresponding upstream license texts. For a complete Rust dependency inventory,
generate one from the locked manifest in an environment with the Rust toolchain, for
example with `cargo metadata` plus a license-audit tool, and archive the output with
the release evidence.

If you distribute a modified build, you are responsible for reviewing the licenses of
the dependencies and tools you use. This notice is informational and is not legal
advice.