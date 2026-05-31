---
title: "B1 — Finding Investigation Issues"
description: "Diagnose Detective finding investigation problems including missing context and incomplete results"
status: active
severity: HIGH
triggers:
  - "finding investigation"
  - "investigate finding"
  - "investigation failed"
  - "no investigation results"
  - "detective investigation"
owner: devops-agent
objective: "Resolve finding investigation issues and ensure complete investigation results"
context: "Detective investigations analyze GuardDuty findings and Security Hub findings to provide context, related entities, and behavioral analysis. Common issues include missing data sources, insufficient data history, and investigation not starting."
---

## Phase 1 — Triage

MUST:
- List investigations: `aws detective list-investigations --graph-arn <graph-arn>`
- Get investigation details: `aws detective get-investigation --graph-arn <graph-arn> --investigation-id <id>`
- List investigation indicators: `aws detective list-indicators --graph-arn <graph-arn> --investigation-id <id>`
- Verify data sources are enabled for the member account

SHOULD:
- Check if the finding's account is a member of the behavior graph
- Verify sufficient data history exists (data takes 24 hours to appear)
- Check investigation status and severity

MAY:
- Start a new investigation: `aws detective start-investigation --graph-arn <graph-arn> --entity-arn <entity-arn> --scope-start-date <start> --scope-end-date <end>`
- Check if the finding type is supported by Detective

## Phase 2 — Remediate

MUST:
- If data sources missing, enable them and wait 24 hours
- If account not a member, add it to the behavior graph
- If investigation incomplete, check data source status for the account

SHOULD:
- Ensure all relevant data sources are enabled (GuardDuty, CloudTrail, VPC Flow Logs)
- Use the correct time scope for the investigation

## Output Format

```yaml
root_cause: "finding_investigation — <specific_cause>"
evidence:
  - type: investigation_status
    content: "<investigation details>"
  - type: indicators
    content: "<investigation indicators>"
severity: HIGH
mitigation:
  immediate: "Fix data sources or account membership"
  long_term: "Ensure all accounts have complete data source coverage"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — Investigations contain security incident details and indicators |
| Blast Radius | LOW — Investigations are read-only analysis |
| Rollback Complexity | LOW — Investigations can be restarted |
| Compliance Scope | CRITICAL — Security investigations support incident response requirements |

## Escalation Conditions

- Escalate IMMEDIATELY to incident response if investigation reveals active compromise
- Escalate to security team if investigation data is incomplete due to missing data sources
- Escalate to AWS Support if investigations fail to start or produce results

## Data Sensitivity

- **Investigation results** contain security indicators — highest sensitivity
- **Entity activity patterns** reveal user and resource behavior
- **Investigation indicators** may contain IP addresses, user identifiers, and API call patterns

## PROHIBITED Actions

- Do NOT share investigation results outside the authorized security team
- Do NOT ignore investigation indicators suggesting active compromise
- Do NOT delete investigations that may be needed for incident response or legal proceedings
- Do NOT start investigations on accounts without proper authorization

## Phase 3 — Rollback

- No rollback needed — investigations are read-only analysis
- If investigation was started with incorrect parameters, start a new one with correct scope
- Preserve investigation results for incident response records

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
