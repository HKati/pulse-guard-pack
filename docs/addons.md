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
```

---

## DPM Minilog
A simple JSONL-based audit log that records key decision points in a pipeline.

```yaml
- name: Log Gitleaks result
  if: always()
  uses: HKati/pulse-guard-pack/.github/actions/dpm-minilog@v1
  with:
    event: gitleaks
    status: ${{ job.status }}
    data: ${{ toJson(steps.gitleaks.outputs) }}
```

---

## Memory Gap Patcher

Marks “gaps” in the DPM timeline with dedicated records, making the event stream easier to analyze.

```yaml
- name: Patch gaps in DPM
  uses: HKati/pulse-guard-pack/.github/actions/memory-gap-patcher@v1
  with:
    source: artifacts/pulse_dpm.jsonl
    threshold_seconds: 90
```

---

## Static Doc-Linker

Checks links in the repository’s documentation and README via CI.  
Configured using `.lychee.toml` and triggered by the workflow:  
`.github/workflows/docs-link-check.yml`.

---

## Example usage in the main workflow

*These blocks are optional — insert them wherever they make sense in your pipeline.*

```yaml
# 1) After Gitleaks — record the audit result
- name: DPM: record gitleaks outcome
  if: always()
  uses: ./.github/actions/dpm-minilog
  with:
    event: "gitleaks"
    status: ${{ job.status }}
    data: ${{ toJson(steps.gitleaks.outputs) }}

# 2) (Optional) health check stabilization before/after deploy
- name: Resonant health confirm
  uses: ./.github/actions/resonant-feedback
  with:
    url: ${{ inputs.health_url }}
    success_pattern: '"status":"ok"'
    tries: 6
    interval_seconds: 10
    consecutive_required: 2

# 3) (Optional) patch missing timeline segments at the end
- name: Patch DPM gaps
  if: always()
  uses: ./.github/actions/memory-gap-patcher
  with:
    source: artifacts/pulse_dpm.jsonl
    threshold_seconds: 90
```
