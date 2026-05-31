---
title: "A3 — Suppression Rules"
description: "Diagnose and configure GuardDuty suppression rule issues"
status: active
severity: MEDIUM
triggers:
  - "suppression rule"
  - "filter not working"
  - "findings not suppressed"
  - "auto-archive rule"
  - "suppression configuration"
owner: devops-agent
objective: "Identify and resolve suppression rule configuration issues"
context: "Suppression rules auto-archive matching findings. Issues arise from incorrect filter criteria, overly broad or narrow rules, or misunderstanding that suppression hides but does not delete findings. Rules use finding attributes as filter criteria."
---

## Phase 1 — Triage

MUST:
- List existing filters: `aws guardduty list-filters --detector-id <detector-id>`
- Get filter details: `aws guardduty get-filter --detector-id <detector-id> --filter-name <filter-name>`
- Verify the filter action is ARCHIVE (suppression) not NOOP
- Check if the finding matches the filter criteria by comparing finding attributes
- List recent findings to see if matching findings are still appearing: `aws guardduty list-findings --detector-id <detector-id> --max-results 10`

SHOULD:
- Test the filter criteria against a known finding to verify it matches
- Check for conflicting or overlapping suppression rules
- Verify the filter criteria syntax is correct

MAY:
- Check CloudTrail for filter creation/modification events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateFilter --max-results 10`
- Review all suppression rules for overly broad criteria

## Phase 2 — Remediate

MUST:
- Fix filter criteria to correctly match the intended findings
- Ensure the filter action is set to ARCHIVE
- Test the updated filter against sample findings

SHOULD:
- Use specific criteria (finding type + resource attributes) rather than broad filters
- Document the purpose and justification for each suppression rule
- Set up periodic review of suppression rules

MAY:
- Create a naming convention for suppression rules
- Export suppression rule configurations for backup

## Common Issues

- symptoms: "Suppression rule created but findings still appear as active"
  diagnosis: "The filter criteria do not match the finding attributes, or the filter action is not ARCHIVE."
  resolution: "Verify filter criteria match the finding's attributes exactly. Ensure action is ARCHIVE."

- symptoms: "Too many findings being suppressed — missing real threats"
  diagnosis: "The suppression rule criteria are too broad, matching legitimate threats."
  resolution: "Narrow the filter criteria to be more specific. Add additional conditions to reduce scope."

- symptoms: "Previously suppressed findings reappearing"
  diagnosis: "The suppression rule was deleted or modified, or new findings with different attributes are generated."
  resolution: "Verify the suppression rule still exists and its criteria match the new findings."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix filter criteria | YELLOW | Filter modification; reversible |
| Set filter action to ARCHIVE | YELLOW | Suppresses findings; reversible |
| Delete overly broad rules | YELLOW | Removes suppression; findings reappear |

## Escalation Conditions

- Suppression rule creation or modification
- Detector configuration changes
- Broad suppression rules that may hide real threats

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-filter` | MEDIUM | Suppression criteria details |
| `list-filters` | LOW | Filter names |
| `list-findings` | MEDIUM | Finding IDs and metadata |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection
- NEVER create suppression rules without documented justification

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Filter criteria fix | Revert via `update-filter` with previous criteria |
| Filter action change | Revert via `update-filter` |
| Filter deletion | Recreate via `create-filter` with previous config |

## Output Format

```yaml
root_cause: "suppression_rule — <specific_cause>"
evidence:
  - type: filter_config
    content: "<get-filter output>"
  - type: finding_match
    content: "<finding attributes vs filter criteria comparison>"
severity: MEDIUM
mitigation:
  immediate: "Fix the suppression rule criteria"
  long_term: "Implement periodic suppression rule review process"
```

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
  - command: "get-findings"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-detectors"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-threat-intel-set"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling GuardDuty to stop alerts"
  - "NEVER suggest archiving findings without investigation"
  - "NEVER expose finding details in public logs"
