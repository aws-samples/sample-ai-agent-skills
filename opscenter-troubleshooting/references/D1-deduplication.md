---
title: "D1 — Deduplication"
description: "Diagnose OpsItem deduplication issues"
status: active
severity: MEDIUM
triggers: ["duplicate OpsItems", "dedup not working", "too many OpsItems"]
owner: devops-agent
objective: "Resolve deduplication issues"
context: "Deduplication uses the /aws/dedup operational data key. Same dedup string = merged OpsItem. No dedup string = new OpsItem every time."
---

## Phase 1 — Triage

MUST:
- Check OpsItem operational data for dedup key: `aws ssm get-ops-item --ops-item-id <id> --query 'OpsItem.OperationalData./aws/dedup'`
- Compare dedup strings across duplicate OpsItems
- Verify dedup string is consistent for same issue type

SHOULD:
- Check EventBridge rule configuration for dedup string
- Review OpsItem source for dedup behavior

MAY:
- Count duplicate OpsItems per issue type

## Phase 2 — Remediate

MUST:
- Add /aws/dedup operational data key with consistent string
- Ensure same issue type uses same dedup string

SHOULD:
- Configure dedup strings in EventBridge rule input transformers
- Document dedup string conventions

MAY:
- Clean up existing duplicate OpsItems

## Common Issues

- symptoms: "Hundreds of OpsItems for the same CloudWatch alarm"
  diagnosis: "No dedup string configured — each alarm event creates new OpsItem."
  resolution: "Add /aws/dedup key with alarm name as value in EventBridge rule."

## Output Format

```yaml
root_cause: "deduplication — <specific_cause>"
evidence:
  - type: dedup_config
    content: "<dedup string presence and consistency>"
severity: MEDIUM
mitigation:
  immediate: "Configure dedup strings"
  long_term: "Standardize dedup conventions across all OpsItem sources"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — deduplication is a data management function
- Service Disruption Risk: LOW — deduplication issues do not affect monitored resources
- Compliance Impact: MEDIUM — duplicate OpsItems can obscure incident tracking

## Escalation Conditions
- Deduplication fails causing excessive duplicate OpsItems
- Deduplication incorrectly merges unrelated OpsItems
- Deduplication rules stop functioning after configuration changes
- High volume of duplicates overwhelms operational teams
- Deduplication key conflicts cause data loss

## Data Sensitivity
- Deduplication keys and rules — INTERNAL
- OpsItem content used for matching — INTERNAL
- Deduplication statistics — INTERNAL
- Rule configurations — INTERNAL
- Never expose deduplication logic that reveals infrastructure patterns

## Prohibited Actions
- Do NOT disable deduplication without a plan to handle increased OpsItem volume
- Do NOT create overly broad deduplication rules that merge unrelated items
- Do NOT ignore deduplication failures
- Do NOT modify deduplication keys without understanding impact on existing items
- Do NOT bypass deduplication for testing without reverting
- Do NOT delete deduplicated OpsItems without checking merged data

## Phase 3 — Rollback
- Restore previous deduplication rules
- Separate incorrectly merged OpsItems
- Re-enable deduplication if disabled
- Verify deduplication is functioning correctly after changes
- Document deduplication rule changes
- Update deduplication configuration based on findings

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
