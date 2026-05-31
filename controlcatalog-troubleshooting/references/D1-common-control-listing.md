---
title: "D1 — Common Control Listing"
description: "Diagnose common control listing issues"
status: active
severity: MEDIUM
triggers: ["common controls", "list common controls", "shared controls"]
owner: devops-agent
objective: "Resolve common control listing issues"
context: "Common controls are AWS-recommended controls mapping to multiple compliance frameworks."
---

## Phase 1 — Triage

MUST:
- List common controls: `aws controlcatalog list-common-controls`
- Check IAM permissions for controlcatalog:ListCommonControls
- Verify filter syntax if using filters

SHOULD:
- Filter by objective: `aws controlcatalog list-common-controls --common-control-filter '{"objectives":[{"arn":"<arn>"}]}'`
- Handle pagination

MAY:
- Compare common controls with your compliance requirements

## Phase 2 — Remediate

MUST:
- Fix IAM permissions
- Fix filter syntax if applicable

SHOULD:
- Map common controls to compliance frameworks

MAY:
- Create implementation plan for common controls

## Common Issues

- symptoms: "list-common-controls returns empty with filter"
  diagnosis: "Filter objective ARN is invalid."
  resolution: "List objectives first to get valid ARNs, then use in filter."

## Output Format

```yaml
root_cause: "common_control_listing — <specific_cause>"
evidence:
  - type: common_controls
    content: "<common control listing results>"
severity: MEDIUM
mitigation:
  immediate: "Fix listing issue"
  long_term: "Map common controls to compliance requirements"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Common controls are public AWS reference data |
| Blast Radius | LOW — Read-only listing |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | HIGH — Common controls map to multiple compliance frameworks |

## Escalation Conditions

- Escalate to compliance team if common control listing gaps affect framework coverage
- Escalate to AWS Support if filter by objective returns unexpected empty results

## Data Sensitivity

- **Common control data** is public reference information
- **Framework mappings** inform compliance decisions — document carefully

## PROHIBITED Actions

- Do NOT assume common controls cover all requirements of any single framework
- Do NOT implement common controls without understanding their specific behavior and scope

## Phase 3 — Rollback

- No rollback needed — read-only listing
- If compliance plans were based on incomplete common control data, reassess coverage

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
