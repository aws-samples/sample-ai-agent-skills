---
title: "Z1 — General Resource Explorer Troubleshooting"
description: "Catch-all runbook for Resource Explorer issues"
status: active
severity: MEDIUM
triggers: ["Resource Explorer error", "resource explorer not working"]
owner: devops-agent
objective: "Systematic investigation of unclassified Resource Explorer issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Check index status: `aws resource-explorer-2 get-index`
- List all indexes: `aws resource-explorer-2 list-indexes`
- Verify credentials: `aws sts get-caller-identity`

SHOULD:
- Test a simple search: `aws resource-explorer-2 search --query-string "*"`
- Check AWS Health Dashboard for service issues

MAY:
- Review CloudTrail for Resource Explorer events
- Check CloudWatch metrics for the service

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up monitoring for recurring issues

## Safety Ratings

```
safety_ratings:
  - "resource-explorer-2 get-index: GREEN — read-only index status"
  - "resource-explorer-2 list-indexes: GREEN — read-only enumeration"
  - "resource-explorer-2 search: GREEN — read-only search"
  - "sts get-caller-identity: GREEN — read-only identity check"
```

## Escalation Conditions

- Resource Explorer service appears non-functional
- All indexes are ACTIVE but search returns no results
- Multiple users experience the same issue simultaneously
- Issue cannot be classified into any specific runbook

## Data Sensitivity

- `aws resource-explorer-2 get-index`: LOW — returns index metadata
- `aws resource-explorer-2 search`: MEDIUM — reveals resource inventory
- `aws sts get-caller-identity`: LOW — reveals account and principal

## Prohibited Actions

- NEVER delete indexes during general troubleshooting
- NEVER make configuration changes without identifying root cause
- NEVER dismiss the issue without documenting the investigation
- NEVER assume search results represent the complete resource inventory

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Verify all indexes are in ACTIVE state
3. Test search functionality after investigation
4. Document all investigation steps and findings
5. Verify Resource Explorer is functioning normally before closing

## Common Issues

- symptoms: "Resource Explorer returns empty results for all queries"
  diagnosis: "No index exists in the current region, or index is in CREATING state."
  resolution: "Check index status with get-index. Create an index if missing. Wait for ACTIVE status."

## Output Format

```yaml
root_cause: "general_resource_explorer — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring"
```

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
