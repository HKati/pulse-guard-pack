# BOXED SECRETS — Safe examples for docs and tests

**Goal:** show realistic-looking values in docs/tests **without** ever
authenticating to real services and **without** triggering secret scanners.

## Principles

1. **Never use live credentials.** Not even revoked ones.
2. **Prefix clearly:** `EXAMPLE_` or `BOXED_` so humans can tell at a glance.
3. **Break the format** so scanners won’t match:
   - wrong length (shorter or longer),
   - safe separators (e.g., `-DEMO`), or
   - change the alphabet (`_` or `*`), so it is **not** a valid token.
4. **Explain that these are placeholders** and must be replaced by owners with
   proper repository/organization secrets.

## Safe patterns (copy‑paste)

- **GitHub token (fake):**
  - `ghp_example_abcdefghijklmnopqrstuvwxyz012345`
  - `github_pat_EXAMPLE_abc1234567890_abc1234567890demo`
- **AWS access key (fake):**
  - `AKIAEXAMPLE1111111111`  <!-- wrong checksum/length -->
- **Private key block (fake):**
  ```text
  -----BEGIN PRIVATE KEY-----
  MIICeAIBADANBgkqhkiG9w0BAQEFAAOCAQ8AMIIB*DEMO*FAKE
  -----END PRIVATE KEY-----
