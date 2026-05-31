---
title: "C2 — Objective Listing"
description: "Diagnose objective listing issues"
status: active
severity: LOW
triggers: ["cannot list objectives", "objectives empty", "objective error"]
owner: devops-agent
objective: "Resolve objective listing issues"
context: "Objectives are specific compliance goals within domains. Listing requires proper permissions and pagination."
---

## Phase 1 — Triage

MUST:
- List objectives: `aws controlcatalog list-objectives`
- Check IAM permissions for controlcatalog:ListObjectives
- Handle pagination for complete results

SHOULD:
- Filter by domain if looking for specific objectives
- List domains first: `aws controlcatalog list-domains`

MAY:
- Check API availability in region

## Phase 2 — Remediate

MUST:
- Fix IAM permissions
- Handle pagination

SHOULD:
- Cache objective list for repeated use

MAY:
- Build objective hierarchy programmatically

## Common Issues

- symptoms: "list-objectives returns partial results"
  diagnosis: "Pagination not handled — only first page returned."
  resolution: "Use --next-token to paginate through all results."

## Output Format

```yaml
root_cause: "objective_listing — <specific_cause>"
evidence:
  - type: api_response
    content: "<objective listing results>"
severity: LOW
mitigation:
  immediate: "Fix listing issue"
  long_term: "Automate objective discovery"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Objectives are public reference data |
| Blast Radius | LOW — Read-only listing |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | LOW — Listing objectives is a discovery operation |

## Escalation Conditions

- Escalate to AWS Support if objective listing consistently fails or returns incomplete results
- Escalate to compliance team if required objectives are not found in the catalog

## Data Sensitivity

- **Objective data** is public AWS reference information
- No sensitive data involved

## PROHIBITED Actions

- Do NOT ignore pagination — partial results may miss critical objectives
- Do NOT assume the objective list is static — check periodically for updates

## Phase 3 — Rollback

- No rollback needed — read-only listing operation

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
