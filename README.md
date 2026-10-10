# binfence

Binary release security gate & mitigation regression auditor for CI/CD pipelines.

[![CI](https://github.com/raidshadowmc-sudo/binfence/actions/workflows/ci.yml/badge.svg)](https://github.com/raidshadowmc-sudo/binfence/actions/workflows/ci.yml)
[![Crates.io](https://img.shields.io/crates/v/binfence.svg)](https://crates.io/crates/binfence)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Release](https://img.shields.io/github/v/release/raidshadowmc-sudo/binfence?include_prereleases)](https://github.com/raidshadowmc-sudo/binfence/releases)
[![Powered by: binlens](https://img.shields.io/badge/engine-binlens-brightgreen)](https://github.com/raidshadowmc-sudo/binlens)

`binfence` audits compiled release assets (PE, ELF, Mach-O) in CI/CD before they hit production. It guards against accidental compiler or linker regressions that strip exploit mitigations, violate `W^X` memory protection, corrupt Authenticode digests, or inject abnormal entropy.

---

## Ecosystem: The Binary Security Suite (`binlens` ↔ `binfence`)

`binfence` and [`binlens`](https://github.com/raidshadowmc-sudo/binlens) are sister projects designed to form a complete lifecycle for binary security auditing:

```text
                           ┌────────────────────────┐
                           │      Target Binary     │
                           │   (PE / ELF / Mach-O)  │
                           └───────────┬────────────┘
                                       │
            ┌──────────────────────────┴──────────────────────────┐
            ▼                                                     ▼
┌──────────────────────────────┐                       ┌──────────────────────────────┐
│           binlens            │                       │           binfence           │
│   Deep Inspection Engine     │                       │    CI/CD Release Gate        │
├──────────────────────────────┤                       ├──────────────────────────────┤
│ • Interactive CLI triage     │                       │ • Automated GitHub Actions   │
│ • Shannon entropy heatmaps   │                       │ • Baseline regression audit  │
│ • Rich Header & compiler IDs │   shared core lib     │ • Hard policy enforcement    │
│ • Import / Export resolution │◄─────────────────────►│ • SARIF v2.1.0 generation     │
│ • Disassembly & YARA scans   │    (binlens crate)    │ • GITHUB_STEP_SUMMARY tables │
│ • Multi-format diffing       │                       │ • Non-zero exit code on fail │
└──────────────┬───────────────┘                       └──────────────┬───────────────┘
               │                                                      │
               ▼                                                      ▼
   Developer / Analyst Triage                              Production Release Gate
   (Interactive Investigation)                              (Automated CI Enforcement)
```

### When to Use Which Tool?

| Task / Persona | Recommended Tool | Command / Action |
|---|---|---|
| **Investigating an unknown binary or malware** | `binlens` | `binlens scan ./suspicious.exe` |
| **Visualizing entropy & unpacking stubs** | `binlens` | `binlens entropy ./sample.bin` |
| **Comparing two local builds interactively** | `binlens` | `binlens diff ./v1.exe ./v2.exe` |
| **Enforcing policy gates on release builds** | `binfence` | `binfence --binary ./target/release/app` |
| **Detecting mitigation regression vs baseline in CI** | `binfence` | `binfence --binary ./new --baseline ./old` |
| **Publishing security alerts to GitHub Security tab** | `binfence` | `uses: raidshadowmc-sudo/binfence@v0.1.3` |

### Synergy: Triage Workflow When a Gate Fails
When `binfence` halts a GitHub Actions pipeline with a mitigation drop or an unexpected RWX section:
1. CI uploads the SARIF report to GitHub Security and fails the job with a descriptive Step Summary.
2. The developer pulls the baseline asset and reproduces the failure locally with full forensic visibility:
   ```bash
   # Pinpoint exact mitigation drift and structural changes
   binlens diff ./old_release ./new_release

   # Deep-dive into sections, headers, and imported APIs
   binlens scan ./new_release
   ```

---

## Why binfence?

Linter and dependency checkers (`cargo audit`, `npm audit`, CodeQL) audit source code and dependency lockfiles. **Almost nobody audits the compiled binary artifact itself.**

A routine dependency bump or toolchain update can silently:
* Drop **ASLR / PIE** layout randomization.
* Disable **DEP / NX** stack protection.
* Introduce simultaneously writable and executable (**RWX**) sections violating `W^X`.
* Invalidate or tamper with **Authenticode** signatures post-build.
* Inflate Shannon entropy through unexpected packing or crypto payloads.

`binfence` acts as the final gate between your build pipeline and your GitHub Releases.

---

## GitHub Action Usage

Add `binfence` to your build or release workflow:

```yaml
name: Release Gate

on:
  release:
    types: [published]
  push:
    tags: ['v*']

permissions:
  contents: read
  security-events: write

jobs:
  audit-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build production binary
        run: cargo build --release

      - name: Run Binary Release Gate
        uses: raidshadowmc-sudo/binfence@v0.1.3
        with:
          binary: target/release/my_app
          fail-on-degraded: true
          fail-on-rwx: true
          require-aslr: true
          require-dep: true

      - name: Upload SARIF to GitHub Security tab
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: binfence.sarif
```

### Comparing Against Baseline (Differential Gate)

To catch regressions between consecutive releases:

```yaml
- name: Download Previous Release Asset
  run: gh release download v1.0.0 --pattern "my_app" --output old_app
  env:
    GH_TOKEN: ${{ github.token }}

- name: Audit Against Baseline
  uses: raidshadowmc-sudo/binfence@v0.1.3
  with:
    binary: target/release/my_app
    baseline: old_app
    fail-on-degraded: true
```

---

## GitHub Step Summary Output

`binfence` automatically posts a rich Markdown table into `$GITHUB_STEP_SUMMARY`:

```markdown
### 🟢 Binfence Security Gate: PASSED

| Metric | Target Asset | Baseline (Previous) | Status |
| :--- | :--- | :--- | :--- |
| **Binary Name** | `my_app` | `old_app` | - |
| **Format / Arch** | ELF64 / x86_64 | ELF64 / x86_64 | - |
| **File Size** | 1,420,896 bytes | 1,398,120 bytes | +22,776 bytes |
| **Shannon Entropy** | 6.210 / 8.000 | 6.195 | 🟢 Normal |
| **ASLR / PIE** | 🟢 Enabled | 🟢 Enabled | ⚪ Unchanged |
| **DEP / NX** | 🟢 Enabled | 🟢 Enabled | ⚪ Unchanged |
| **W^X (No RWX)** | 🟢 Enforced | 🟢 Enforced | 🟢 OK |
| **Authenticode** | ⚪ Unsigned | ⚪ Unsigned | ⚪ |
```

---

## CLI Usage

`binfence` can also be run locally on your development machine:

### Prebuilt Binaries (Linux, macOS, Windows)
Standalone pre-compiled binaries are published for every release on [GitHub Releases](https://github.com/raidshadowmc-sudo/binfence/releases/latest):

| Target Platform | Architecture | Binary Package |
|---|---|---|
| **Linux** | `x86_64` (glibc) | [`binfence-*-x86_64-unknown-linux-gnu.tar.gz`](https://github.com/raidshadowmc-sudo/binfence/releases/latest) |
| **Linux** | `aarch64` (ARM64) | [`binfence-*-aarch64-unknown-linux-gnu.tar.gz`](https://github.com/raidshadowmc-sudo/binfence/releases/latest) |
| **Windows** | `x86_64` (MSVC) | [`binfence-*-x86_64-pc-windows-msvc.zip`](https://github.com/raidshadowmc-sudo/binfence/releases/latest) |
| **macOS** | Apple Silicon (`aarch64`) | [`binfence-*-aarch64-apple-darwin.tar.gz`](https://github.com/raidshadowmc-sudo/binfence/releases/latest) |
| **macOS** | Intel (`x86_64`) | [`binfence-*-x86_64-apple-darwin.tar.gz`](https://github.com/raidshadowmc-sudo/binfence/releases/latest) |

### From crates.io
```bash
# Install from crates.io
cargo install binfence --locked

# Or directly from Git repository
cargo install --git https://github.com/raidshadowmc-sudo/binfence.git --locked

# Audit binary against default security policies
binfence --binary ./app.exe

# Differential audit against previous build
binfence --binary ./build/new_app.exe --baseline ./build/old_app.exe

# Enforce strict policy flags
binfence --binary ./dist/app \
  --fail-on-degraded \
  --require-aslr \
  --require-dep \
  --disallow-rwx \
  --fail-on-tampered \
  --max-entropy 7.2 \
  --sarif-file release.sarif
```

---

## Security Rules Evaluated

| Rule ID | Name | Severity | Description |
| :--- | :--- | :--- | :--- |
| `BIN001-*` | `SecurityMitigationDegraded` | 🔴 Error | An enabled mitigation (`ASLR`, `DEP`, `CFG`, `RELRO`, `Canary`, `FORTIFY`, `SafeSEH`) was dropped compared to baseline. |
| `BIN002-NO-ASLR` | `RequireASLR` | 🔴 Error | Binary was linked without dynamic base or position-independent executable (PIE) support. |
| `BIN003-NO-DEP` | `RequireDEP` | 🔴 Error | Binary does not enforce non-executable stack/heap protection (`NX_COMPAT` or `PT_GNU_STACK`). |
| `BIN004-RWX-SECTION` | `DisallowRWXSections` | 🔴 Error | Binary contains simultaneously writable and executable sections (`W^X` violation). |
| `BIN005-AUTHENTICODE-TAMPERED` | `AuthenticodeDigestMismatch` | 🔴 Error | PE Authenticode digest does not match embedded signature, indicating binary tampering. |
| `BIN005-AUTHENTICODE-MALFORMED`| `AuthenticodeMalformed` | 🔴 Error | Embedded PKCS#7 certificate directory is corrupt or malformed. |
| `BIN006-UNSIGNED-BINARY` | `RequireAuthenticode` | 🔴 Error | Policy requires an embedded Authenticode signature, but binary is unsigned (opt-in for PE). |
| `BIN007-HIGH-ENTROPY` | `HighEntropyWarning` | ⚠️ Warning | Shannon entropy exceeds threshold (default: 7.5), signaling possible packing or hidden payload. (Emits advisory warning; does not block release gate). |
| `BIN008-YARA-*` | `YaraSignatureMatch` | 🔴 Error | Target binary matched a prohibited YARA signature rule. |
| `BIN009-NO-CFG` | `RequireControlFlowGuard` | 🔴 Error | Policy requires Control Flow Guard (`/guard:cf`) on Windows PE binaries. |
| `BIN010-NO-STACK-CANARY` | `RequireStackCanary` | 🔴 Error | Policy requires stack smash buffer security checks (`/GS` or `-fstack-protector`). |
| `BIN011-EXCESS-NEW-SECTIONS` | `ExcessiveNewSections` | 🔴 Error | Number of newly added sections exceeds allowable threshold. |

---

## License

Licensed under the [MIT License](LICENSE).
