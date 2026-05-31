---
title: "D2 — Export Output Issues"
description: "Diagnose journal export output format and content problems"
status: active
severity: MEDIUM
triggers: ["export empty", "Ion format", "cannot parse export"]
owner: devops-agent
objective: "Identify and resolve export output issues"
---

## Phase 1 — Triage

MUST:
- Verify export files exist in S3
- Check export format (Ion)
- Verify time range includes expected data
- Check export manifest file

SHOULD:
- Use Ion libraries to parse export data
- Verify export includes expected tables

## Phase 2 — Remediate

MUST:
- Use Ion parser for export data (not JSON parser)
- Adjust time range to include expected data
- Check manifest for export completeness

## Output Format

```yaml
root_cause: "export_output — <specific_cause>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — export output contains ledger data
- Service Disruption Risk: LOW — export issues do not affect ledger operations
- Compliance Impact: HIGH — export data must comply with data handling policies

## Escalation Conditions
- Export output corrupted or incomplete
- Export output contains sensitive data that should be redacted
- S3 output permissions misconfigured
- Export output format incompatible with downstream systems
- Export output storage costs exceed expectations

## Data Sensitivity
- Exported ledger data — CONFIDENTIAL
- S3 output paths — INTERNAL
- Export configurations — INTERNAL
- Data format specifications — INTERNAL
- Never expose exported ledger data to unauthorized parties

## Prohibited Actions
- Do NOT store export output in publicly accessible S3 buckets
- Do NOT disable S3 encryption for export output
- Do NOT share export output with unauthorized parties
- Do NOT retain export data beyond approved retention periods
- Do NOT ignore export output validation errors
- Do NOT grant broad S3 access to export roles

## Phase 3 — Rollback
- Delete incorrectly exported data
- Restore S3 bucket configurations
- Revert export role permissions
- Re-run export with corrected configurations
- Verify export output integrity
- Document export issues and data handling procedures

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
