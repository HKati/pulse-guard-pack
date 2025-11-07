# Add-ons (Optional)

These modules do not change the core secret-protection logic but improve
developer experience (DX) and audit visibility.

## Resonant Feedback Hook
Filters flapping status transitions by polling a health endpoint multiple times.

**Usage (consumer repo):**
```yaml
- name: Confirm health
  uses: HKati/pulse-guard-pack/.github/actions/resonant-feedback@v1
  with:
    url: https://status.example/health
    success_pattern: '"status":"ok"'
    tries: 5
    interval_seconds: 10
    consecutive_required: 2
