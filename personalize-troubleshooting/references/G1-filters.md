---
title: "G1 — Filter Configuration Issues"
description: "Diagnose recommendation filter problems"
status: active
severity: MEDIUM
triggers: ["filter not working", "filter expression error", "CreateFilter error"]
owner: devops-agent
objective: "Identify and resolve filter configuration issues"
context: "Filters use SQL-like expressions to include/exclude items from recommendations. Expressions reference item/user metadata. Validated at creation time."
---

## Phase 1 — Triage

MUST:
- List filters: `aws personalize list-filters --dataset-group-arn <arn>`
- Check filter expression syntax
- Verify referenced fields exist in schema
- Test recommendations with and without filter

SHOULD:
- Check filter expression against item metadata
- Verify dynamic filter parameters if used
- Test with known items that should be filtered

MAY:
- Compare filtered vs unfiltered results
- Check for filter quota limits

## Phase 2 — Remediate

MUST:
- Use correct filter expression syntax (INCLUDE/EXCLUDE ItemID WHERE...)
- Reference existing schema fields
- Create Items dataset with metadata for filtering

SHOULD:
- Test filter expressions with sample data
- Use dynamic parameters for runtime filtering

MAY:
- Implement filter management pipeline

## Common Issues

- symptoms: "CreateFilter fails with expression error"
  diagnosis: "Invalid filter expression syntax."
  resolution: "Use correct syntax: EXCLUDE ItemID WHERE Items.FIELD IN (\"value\")."

- symptoms: "Filter doesn't exclude expected items"
  diagnosis: "Item metadata missing or field name mismatch."
  resolution: "Verify Items dataset has the referenced field. Check field name case."

## Output Format

```yaml
root_cause: "filters — <specific_cause>"
evidence:
  - type: filter_config
    content: "<filter expression and details>"
severity: MEDIUM
mitigation:
  immediate: "Fix filter expression"
  long_term: "Implement filter testing"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — filters control recommendation output
- Service Disruption Risk: MEDIUM — filter issues can serve inappropriate recommendations
- Compliance Impact: HIGH — filters enforce content and compliance policies

## Escalation Conditions
- Filters fail to exclude prohibited content
- Filter expressions cause recommendation API errors
- Filters too restrictive causing empty recommendation results
- Filter changes affect recommendation quality
- Compliance-required filters accidentally removed

## Data Sensitivity
- Filter expressions — INTERNAL
- Filtered content categories — INTERNAL
- Filter evaluation metrics — INTERNAL
- Compliance filter configurations — CONFIDENTIAL
- Never expose filter logic that reveals content policies

## Prohibited Actions
- Do NOT remove compliance-required filters without approval
- Do NOT deploy filters without testing
- Do NOT create filters that discriminate based on protected characteristics
- Do NOT ignore filter evaluation errors
- Do NOT bypass content filtering for any reason
- Do NOT share filter configurations externally

## Phase 3 — Rollback
- Restore previous filter configurations
- Re-enable removed compliance filters
- Verify filter effectiveness after changes
- Restore filter expressions to previous versions
- Document filter changes and compliance impact
- Update filter management procedures

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
