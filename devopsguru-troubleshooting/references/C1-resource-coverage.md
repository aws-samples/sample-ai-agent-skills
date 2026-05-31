---
title: "C1 — Resource Coverage Issues"
description: "Diagnose resource coverage configuration and gaps"
status: active
severity: HIGH
triggers: ["resources not monitored", "coverage gap", "missing resources"]
owner: devops-agent
objective: "Resolve resource coverage issues"
context: "Coverage via CloudFormation stacks, tags, or account-wide."
---

## Phase 1 — Triage

MUST:
- Check coverage: `aws devops-guru describe-resource-collection-health --resource-collection-type AWS_SERVICE`
- Verify coverage configuration method
- Check for uncovered resources

SHOULD:
- Review tag-based coverage rules
- Check CloudFormation stack coverage

## Phase 2 — Remediate

MUST:
- Add missing resources to coverage
- Fix tag-based coverage rules
- Verify coverage after changes

## Common Issues

- symptoms: "Some resources not generating insights"
  diagnosis: "Resources not included in coverage."
  resolution: "Add resources via tags or CloudFormation stacks."

## Output Format

```yaml
root_cause: "resource_coverage — <specific_cause>"
evidence:
  - type: coverage_config
    content: "<coverage configuration>"
severity: HIGH
mitigation:
  immediate: "Fix coverage gaps"
  long_term: "Automate coverage management"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Coverage configuration is operational metadata |
| Blast Radius | HIGH — Coverage gaps leave resources unmonitored |
| Rollback Complexity | LOW — Coverage can be updated immediately |
| Compliance Scope | MEDIUM — Complete coverage supports availability requirements |

## Escalation Conditions

- Escalate to operations if critical resources are not covered
- Escalate to management if coverage gaps affect SLA compliance

## Data Sensitivity

- **Resource coverage** reveals which resources are monitored — operational data
- **Coverage gaps** indicate monitoring blind spots

## PROHIBITED Actions

- Do NOT remove critical production resources from coverage
- Do NOT enable account-wide coverage without understanding cost implications

## Phase 3 — Rollback

- Restore previous coverage configuration
- Re-add resources that were removed from coverage
- Verify coverage health after changes

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
