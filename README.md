[![Secret leak guard](https://github.com/HKati/pulse-guard-pack/actions/workflows/secret-leak-guard.yml/badge.svg?branch=main)](https://github.com/HKati/pulse-guard-pack/actions/workflows/secret-leak-guard.yml)
[![Preflight secret scan](https://github.com/HKati/pulse-guard-pack/actions/workflows/addons-selftest.yml/badge.svg?branch=main)](https://github.com/HKati/pulse-guard-pack/actions/workflows/addons-selftest.yml)




# Pulse Guard Pack — boxed secrets & secret‑leak guard



<p align="center">
  <img src="docs/hero.png"
       alt="Pulse Guard Pack — boxed secrets & secret-leak guard (PULSE)"
       width="900">

> **Purpose.** Prevent accidental API‑key commits with minimal friction.  
> **Who for.** Any team adopting CI/CD (with or without GitHub Advanced Security).  
> **Why now.** Secret leaks happen even to great teams. Shipping a *default‑sane*, opt‑in guard reduces risk without slowing velocity.

<!-- Feature strip (compatibility & policy) -->

See **Add-ons (Optional)** guide → [`docs/addons.md`](./docs/addons.md)

[![Secret leak guard](https://github.com/HKati/pulse-guard-pack/actions/workflows/secret-leak-guard.yml/badge.svg?branch=main)](https://github.com/HKati/pulse-guard-pack/actions/workflows/secret-leak-guard.yml)
[![Preflight secret scan](https://github.com/HKati/pulse-guard-pack/actions/workflows/addons-selftest.yml/badge.svg?branch=main)](https://github.com/HKati/pulse-guard-pack/actions/workflows/addons-selftest.yml)

[![license: Apache-2.0](https://img.shields.io/badge/license-Apache--2.0-blue)](LICENSE)
![works with GitHub](https://img.shields.io/badge/works%20with%20GitHub-informational)
![works with GitLab](https://img.shields.io/badge/works%20with%20GitLab-informational)
![works with Jenkins](https://img.shields.io/badge/works%20with%20Jenkins-informational)
![works with Azure Pipelines](https://img.shields.io/badge/works%20with%20Azure%20Pipelines-informational)
[![security policy](https://img.shields.io/badge/security-policy-blue)](SECURITY.md)

---

## Features at a glance

- **Reusable GitHub Action**: one‑line integration; no GHAS required; redacted logs.
- **Boxed secrets** pattern: real keys live in **Environments**, *not in the repo*.
- **Pre‑commit hooks** (optional) for developer‑side prevention.
- **Low noise**: minimal allowlist for tests/examples; CI finishes in seconds.
- **Portable**: GitLab/Jenkins/Azure snippets included.

---

## Quickstart (GitHub Actions — 1 line)

> Paste this into a workflow file in your repo (e.g. `.github/workflows/guard.yml`).

~~~yaml
name: Example - Pulse Guard Pack

on:
  pull_request:
  push:
    branches: [ main ]

permissions:
  contents: read
  actions: read            # needed on private repos for SARIF upload metadata
  security-events: write   # needed to upload SARIF to Code Scanning

jobs:
  guard:
    uses: HKati/pulse-guard-pack/.github/workflows/secret-leak-guard.yml@v1
# with:
#   extra_args: "--config-path .gitleaks.toml"

~~~

> **Note:** `security-events: write` is only required if you upload SARIF to Code Scanning.  
> For fast default runs, it is not needed.

---

#### CI-ergonomics (optional)

To avoid duplicate concurrent runs on the same branch or pull request, add a concurrency guard to your workflow:

~~~yaml
concurrency:
  group: guard-${{ github.ref }}
  cancel-in-progress: true
~~~

---

## Configuration (optional but recommended)

### `.gitleaks.toml` (false‑positive trimming)
Place in repo root:

    title = "Pulse guard pack — gitleaks config"

    [allowlist]
    paths = [
      "tests/fixtures/",
      "docs/examples/",
      "examples/"
    ]
    regexes = [
      "(?i)BEGIN PUBLIC KEY",      # public keys are not secrets
      "(?i)localhost(:\\d+)?",
      "(?i)example\\.com"
    ]

Then pass it via `extra_args: "--config-path .gitleaks.toml"`.

#### Triage quick-guide (false positives)

- **Finding → Verify** the match and file path.
- If benign, **add a targeted allowlist** (regex + path) in `.gitleaks.toml`.
- **Never** use broad/global allows; keep them scoped to tests/examples only.


### Developer‑side prevention (pre‑commit)

`.pre-commit-config.yaml`:

    repos:
      - repo: https://github.com/zricethezav/gitleaks
        rev: v8.18.2
        hooks:
          - id: gitleaks
            args: ["--redact", "--no-git", "-s", "."]

      - repo: https://github.com/Yelp/detect-secrets
        rev: v1.4.0
        hooks:
          - id: detect-secrets
            args: ["--baseline", ".secrets.baseline"]

Install:

    pip install pre-commit
    pre-commit install
    detect-secrets scan > .secrets.baseline
    git add .secrets.baseline

### Recommended GitHub rules

- **Require PR before merge** (≥1 reviewer).  
- **Required status checks**: `Secret leak guard`.  
- **Block force pushes** on default branch.  
- **Restrict file paths** (Block):  
  `*.pem`, `*.key`, `.env`, `*.p12`, `*_sa.json`, `*credentials*.json`, `id_rsa`.

### Boxed secrets (where real keys live)

- Store real keys only in **Environments** (e.g., `staging`, `prod`).  
- Inject as `${{ secrets.YOUR_KEY }}` at runtime.  
- Never commit `.env` files; keep examples as `.env.example` with placeholders only.

> Note: `security-events: write` is only required if you upload SARIF to
> Code Scanning. For fast runs, keep `scan-history: false`; set it to `true`
> to scan the full git history (slower).

---

## Advanced (optional)

### Upload to Code Scanning (SARIF) — public repos with GHAS

    jobs:
      guard:
        uses: eplabsai/pulse-guard-pack/.github/workflows/secret_guard.yml@v1
        with:
          extra_args: "--report-format sarif --report-path gitleaks.sarif"
      upload:
        needs: guard
        if: ${{ always() }}
        runs-on: ubuntu-latest
        permissions:
          security-events: write
          contents: read
        steps:
          - uses: actions/checkout@v4
          - name: Upload SARIF
            uses: github/codeql-action/upload-sarif@v3
            with:
              sarif_file: gitleaks.sarif

### Optional runtime masking (if secrets don’t come from `secrets.*`)

    - name: Mask runtime secrets
      run: |
        for v in FOO_TOKEN BAR_API_KEY; do
          [ -n "${!v:-}" ] && echo "::add-mask::${!v}"
        done
      env:
        FOO_TOKEN: ${{ secrets.FOO_TOKEN }}
        BAR_API_KEY: ${{ secrets.BAR_API_KEY }}

---

## Other CI platforms

### GitLab CI

    secret_leak_guard:
      image: zricethezav/gitleaks:latest
      stage: test
      script:
        - gitleaks detect --redact --no-git -s . --exit-code 1
      only:
        - merge_requests
        - main

### Jenkins (Declarative Pipeline)

    pipeline {
  agent any
  stages {
    stage('Secret leak guard') {
      steps {
        sh '''
          set -euo pipefail
          ARCH=$(uname -m)
          case "$ARCH" in
            x86_64|amd64) ASSET=linux_x64 ;;
            aarch64|arm64) ASSET=linux_arm64 ;;
            *) echo "Unsupported arch: $ARCH" >&2; exit 2 ;;
          esac
          curl -sSL "https://github.com/zricethezav/gitleaks/releases/latest/download/gitleaks_${ASSET}.tar.gz" | tar -xz
          ./gitleaks detect --redact --no-git -s . --exit-code 1
        '''
      }
    }
  }
}


### Azure Pipelines

    trigger:
      branches: { include: [ main ] }
    pr:
      branches: { include: [ main ] }

    pool: { vmImage: ubuntu-latest }

    steps:
      - script: |
          curl -sSL https://github.com/zricethezav/gitleaks/releases/latest/download/gitleaks_linux_x64.tar.gz | tar -xz
          ./gitleaks detect --redact --no-git -s . --exit-code 1
        displayName: Secret leak guard

---

## Security & privacy notes

- Logs are **redacted**; the workflow uploads **no artifacts** by default.  
- The guard scans the **working tree**; no network access needed.  
- This guard **complements**, not replaces, GHAS Secret Scanning & Push Protection.

---

## Versioning & compatibility

- Use the **major tag**: `@v1` for stability.  
- SemVer for releases (`v1.0.0`, `v1.1.0`, …).  
- Compatible with Ubuntu runners on GitHub Actions; other CI via binaries or Docker.

---

## FAQ

**Q: Will this block my merge?**  
A: Yes—if a likely secret is present, the job exits non‑zero so required‑checks fail. Make it required in branch rules to enforce.

**Q: False positives?**  
A: Trim with a scoped `.gitleaks.toml` (tests/examples only) and review via CODEOWNERS.

**Q: Do I need GH Advanced Security?**  
A: No. This works without GHAS. With GHAS, you can also upload SARIF to Code Scanning.

---

## Troubleshooting

- **“It flagged a public key.”** Add `(BEGIN PUBLIC KEY)` to the allowlist regex (see sample).  
- **“Noise on docs‑only changes.”** Keep the guard; it’s fast. If absolutely needed, add path filters in the *caller* workflow.  
- **“We use long‑lived cloud keys.”** Prefer OIDC/Federated credentials to avoid storing static keys.

---

## License

Apache License 2.0. See `LICENSE`.

---

## Acknowledgments

- Built on **Gitleaks** (zricethezav) and **detect‑secrets** (Yelp).  
- Part of the **EPLabsAI** PULSE ecosystem (deterministic, fail‑closed release gates).

---

### Why this exists (opinion)

Secrets leak most often by accident. A lightweight guard that is default-sane, deterministic, and fast removes an entire class of incidents without taxing teams.  
This pack keeps risk low and adoption low-friction: one line on GitHub; portable elsewhere.  
Logs are redacted, no repository secrets are required, and defaults are safe for everyday use.

---

### Copy-paste summary

~~~yaml
jobs:
  guard:
    uses: HKati/pulse-guard-pack/.github/workflows/secret-leak-guard.yml@v1
    # Optional: pin to a commit SHA for regulated environments
    # uses: HKati/pulse-guard-pack/.github/workflows/secret-leak-guard.yml@<commit-sha>
~~~

## Maintainers & Contact

Maintained by EPLabsAI.

- 🐞 Issues & feature requests: [GitHub Issues](https://github.com/HKati/pulse-guard-pack/issues)
- 🔐 Security: see [SECURITY.md](SECURITY.md) and use GitHub Private vulnerability reporting.







