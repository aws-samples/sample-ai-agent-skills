---
title: "D1 — GuardDuty Findings Integration"
description: "Diagnose Detective integration issues with GuardDuty findings"
status: active
severity: HIGH
triggers:
  - "guardduty findings detective"
  - "findings not in detective"
  - "guardduty integration"
  - "finding not showing"
  - "detective guardduty"
owner: devops-agent
objective: "Resolve GuardDuty findings integration issues with Detective"
context: "GuardDuty findings are the primary data source for Detective. Findings should automatically appear in Detective for investigation. Common issues include GuardDuty not enabled, member account not in graph, and data source package not enabled."
---

## Phase 1 — Triage

MUST:
- Verify GuardDuty is enabled: `aws guardduty list-detectors`
- Check GuardDuty findings exist: `aws guardduty list-findings --detector-id <id> --max-results 5`
- Verify Detective DETECTIVE_CORE data source is enabled: `aws detective list-datasource-packages --graph-arn <graph-arn>`
- Check if the finding's account is a member of the behavior graph

SHOULD:
- Verify the finding is in the same region as the behavior graph
- Check if 24 hours have passed since the finding was generated
- Verify GuardDuty finding is not archived or suppressed

MAY:
- Check CloudTrail for Detective data ingestion events
- Verify no service issues affecting data flow

## Phase 2 — Remediate

MUST:
- If GuardDuty not enabled, enable it and wait 48 hours
- If account not a member, add to behavior graph
- If data source not enabled, enable DETECTIVE_CORE package
- Wait 24 hours for data to appear

SHOULD:
- Enable all GuardDuty protection plans for comprehensive findings
- Verify findings are not being suppressed by GuardDuty filters

## Output Format

```yaml
root_cause: "guardduty_integration — <specific_cause>"
evidence:
  - type: guardduty_status
    content: "<GuardDuty detector status>"
  - type: datasource_status
    content: "<Detective data source status>"
severity: HIGH
mitigation:
  immediate: "Fix GuardDuty enablement or data source configuration"
  long_term: "Ensure GuardDuty and Detective are enabled in all regions"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — GuardDuty findings contain security threat intelligence |
| Blast Radius | MEDIUM — Missing findings integration leaves security gaps |
| Rollback Complexity | LOW — Enable data source and wait for ingestion |
| Compliance Scope | CRITICAL — GuardDuty-Detective integration is core to security monitoring |

## Escalation Conditions

- Escalate IMMEDIATELY to security if GuardDuty findings are not reaching Detective
- Escalate to incident response if findings indicate active threats but cannot be investigated
- Escalate to AWS Support if data ingestion fails despite correct configuration

## Data Sensitivity

- **GuardDuty findings** contain threat intelligence — highest security sensitivity
- **Finding details** include IP addresses, API calls, and resource identifiers

## PROHIBITED Actions

- Do NOT suppress GuardDuty findings without security team review
- Do NOT disable GuardDuty while Detective depends on it
- Do NOT archive findings before they are investigated in Detective

## Phase 3 — Rollback

- Re-enable DETECTIVE_CORE data source if it was disabled
- Re-enable GuardDuty if it was disabled
- Wait 24 hours for data to appear after re-enabling
- Document any gap in security monitoring coverage

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
  - "NEVER suggest disabling Detective to reduce costs"
  - "NEVER suggest ignoring investigation findings"
  - "NEVER expose investigation details publicly"
