# D1 — Data Export Issues

## Symptoms
- Export task fails
- Export file empty or incomplete
- Can't download export

## Triage Commands

```bash
aws discovery describe-export-tasks --max-results 10
aws discovery start-export-task
aws discovery describe-export-tasks --export-ids <export-id>
```

## Common Causes

1. **No data to export** — Collection not started or no data yet
2. **Export in progress** — Large datasets take time
3. **S3 permissions** — Service can't write to export bucket
4. **Export limit** — Concurrent export limit reached
5. **Filter too restrictive** — Filters excluding all data

## Resolution

- Verify data exists: `aws discovery get-discovery-summary`
- Start export: `aws discovery start-export-task`
- Check export status: look for `exportStatus: SUCCEEDED`
- Download from URL in export task response (pre-signed S3 URL)
- For filtered export, verify filter values match existing data


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Verify data exists | GREEN | Read-only check |
| Start export task | GREEN | Non-destructive data export |
| Check export status | GREEN | Read-only check |
| Download from pre-signed URL | GREEN | Data retrieval |
| Verify filter values | GREEN | Read-only check |

## Escalation Conditions

- Discovery agent deployed in production environment

## Data Sensitivity

- **Exported data**: Server inventory, network, performance data — CONFIDENTIAL
- **Pre-signed S3 URLs**: Temporary access to exported data — CONFIDENTIAL
- **Export configuration**: Filter criteria reveal scope of discovery — INTERNAL

## Prohibited Actions

- NEVER deploy discovery agents without change management approval
- NEVER store IAM access keys in plain text on shared systems
- NEVER modify agent configuration on production servers without a rollback plan

## Phase 3 — Rollback

1. Cancel any in-progress export tasks if possible
2. Delete exported data from S3 if it contains sensitive information
3. Revoke pre-signed URLs by deleting the S3 objects
4. Review and restrict S3 bucket permissions
5. Document what data was exported and to whom

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
