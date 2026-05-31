---
title: "A2 — Filter Controls"
description: "Diagnose control filtering issues"
status: active
severity: LOW
triggers: ["filter not working", "wrong controls returned", "filter syntax"]
owner: devops-agent
objective: "Resolve control filtering issues"
context: "Controls can be filtered by domain, objective, and other criteria. Incorrect filters return empty or unexpected results."
---

## Phase 1 — Triage

MUST:
- Verify filter syntax matches API specification
- Check if filter values (domain ARN, objective ARN) are valid
- Try listing without filters to verify API access

SHOULD:
- List available domains: `aws controlcatalog list-domains`
- List available objectives: `aws controlcatalog list-objectives`

MAY:
- Test with different filter combinations

## Phase 2 — Remediate

MUST:
- Fix filter syntax
- Use valid domain/objective ARNs

SHOULD:
- Verify filter values exist before using

MAY:
- Build filter queries programmatically

## Common Issues

- symptoms: "Filter returns empty results but controls exist"
  diagnosis: "Filter value (domain/objective ARN) is incorrect."
  resolution: "List domains/objectives first, then use valid ARNs in filters."

## Output Format

```yaml
root_cause: "filter_controls — <specific_cause>"
evidence:
  - type: filter_config
    content: "<filter syntax and results>"
severity: LOW
mitigation:
  immediate: "Fix filter syntax"
  long_term: "Document valid filter values"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Filter operations on public catalog data |
| Blast Radius | LOW — Read-only filtering; no changes |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | LOW — Filtering is a query operation |

## Escalation Conditions

- Escalate to compliance team if filter issues prevent identifying required controls
- Escalate to AWS Support if filter syntax is correct but returns unexpected results

## Data Sensitivity

- **Filter parameters** (domain ARNs, objective ARNs) are public reference identifiers
- No sensitive data involved in filter operations

## PROHIBITED Actions

- Do NOT assume empty filter results mean no controls exist — verify filter syntax first
- Do NOT hardcode control ARNs — they may change across regions

## Phase 3 — Rollback

- No rollback needed — read-only query operation
- Correct filter syntax and retry

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
