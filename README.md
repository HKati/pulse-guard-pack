[![Secret leak guard](https://github.com/<owner>/pulse-guard-pack/actions/workflows/self_test.yml/badge.svg)](…)


# Pulse Guard Pack — boxed secrets & secret‑leak guard

> **Purpose.** Prevent accidental API‑key commits with minimal friction.  
> **Who for.** Any team adopting CI/CD (with or without GitHub Advanced Security).  
> **Why now.** Secret leaks happen even to great teams. Shipping a *default‑sane*, opt‑in guard reduces risk without slowing velocity.

![license](https://img.shields.io/badge/license-Apache--2.0-blue.svg)
![status](https://img.shields.io/badge/gh-actions-reusable%20workflow-brightgreen.svg)
![ecosystem](https://img.shields.io/badge/works%20with-GitHub%20%7C%20GitLab%20%7C%20Jenkins%20%7C%20Azure-6aa84f.svg)

---

## Features at a glance

- **Reusable GitHub Action**: one‑line integration; no GHAS required; redacted logs.
- **Boxed secrets** pattern: real keys live in **Environments**, *not in the repo*.
- **Pre‑commit hooks** (optional) for developer‑side prevention.
- **Low noise**: minimal allowlist for tests/examples; CI finishes in seconds.
- **Portable**: GitLab/Jenkins/Azure snippets included.

---

## Quick start (GitHub Actions — 1 line)

Create: `.github/workflows/secret_leak_guard.yml`

    name: Secret leak guard
    on:
      pull_request:
      push:
        branches: [ main ]
    permissions:
      contents: read
    jobs:
      guard:
        uses: eplabsai/pulse-guard-pack/.github/workflows/secret_guard.yml@v1
        # with:
        #   gitleaks_version: v8.18.2      # optional
        #   extra_args: "--config-path .gitleaks.toml"  # optional

> **Toggle (optional).** If you want a UI switch: add  
> `if: ${{ vars.PULSE_SECRET_GUARD != 'off' }}`  
> to the calling job and set the repo variable in **Settings → Variables**.

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
            sh 'curl -sSL https://github.com/zricethezav/gitleaks/releases/latest/download/gitleaks_$(uname -s)_$(uname -m).tar.gz | tar -xz'
            sh './gitleaks detect --redact --no-git -s . --exit-code 1'
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

## Why this exists (opinion)

Secrets leak most often by accident. A lightweight guard that is **default‑sane**, **deterministic**, and **fast** removes a whole class of incidents without taxing teams. This pack keeps risk low and adoption low‑friction: one line on GitHub; portable elsewhere.

---

## Copy‑paste summary

    jobs:
      guard:
        uses: eplabsai/pulse-guard-pack/.github/workflows/secret_guard.yml@v1
