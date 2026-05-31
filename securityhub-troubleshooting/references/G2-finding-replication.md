---
title: "G2 — Finding Replication Issues"
description: "Diagnose Security Hub finding replication delays and consistency issues"
status: active
severity: MEDIUM
triggers:
  - "finding replication"
  - "finding delay"
  - "finding sync"
  - "stale findings"
  - "finding propagation"
owner: devops-agent
objective: "Identify and resolve finding replication delays and consistency issues across regions and accounts"
context: "Finding replication across regions and accounts is eventually consistent. Delays can occur due to high finding volume, service throttling, or configuration issues. Understanding normal propagation times helps distinguish real issues from expected behavior."
---

## Phase 1 — Triage

MUST:
- Check finding aggregator configuration: `aws securityhub list-finding-aggregators`
- Compare finding counts between source and aggregation regions
- Check finding timestamps (CreatedAt, UpdatedAt) for propagation delays
- Verify Security Hub is enabled and healthy in all regions: `aws securityhub describe-hub`
- Check for service throttling in CloudTrail

SHOULD:
- Compare specific findings between source and aggregation regions
- Check CloudWatch metrics for Security Hub in both regions
- Verify member account association status: `aws securityhub list-members`

MAY:
- Check AWS Health Dashboard for Security Hub service events
- Verify no SCPs are blocking cross-region API calls

## Phase 2 — Remediate

MUST:
- Verify the finding aggregator is properly configured
- Ensure Security Hub is enabled in all source regions
- Allow time for eventual consistency (minutes to hours is normal)

SHOULD:
- Monitor finding propagation times to establish a baseline
- Set up alerts for abnormal propagation delays

MAY:
- Use EventBridge for real-time finding forwarding as a supplement
- Export findings to S3 for guaranteed delivery

## Common Issues

- symptoms: "Findings from source region not appearing in aggregation region"
  diagnosis: "The finding aggregator does not link the source region, or Security Hub is not enabled in the source."
  resolution: "Verify the aggregator links the source region. Enable Security Hub in the source region."

- symptoms: "Finding updates not propagating to aggregation region"
  diagnosis: "Finding updates are eventually consistent. There is a normal delay for status changes."
  resolution: "Allow time for propagation. If persistently delayed, check for service issues."

- symptoms: "Finding counts differ between source and aggregation regions"
  diagnosis: "Propagation delay, finding retention differences, or findings archived in one region."
  resolution: "Compare specific findings rather than counts. Account for retention and archival differences."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Verify aggregator configuration | GREEN | Read-only check |
| Allow propagation time | GREEN | No action needed |

## Escalation Conditions

- Cross-account configuration changes
- Persistent replication delays beyond normal thresholds

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `list-finding-aggregators` | LOW | Aggregator configuration |
| `get-findings` (cross-region) | HIGH | Security findings |
| `describe-hub` | LOW | Hub status |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Aggregator configuration fix | Revert via `update-finding-aggregator` |

## Output Format

```yaml
root_cause: "finding_replication — <specific_cause>"
evidence:
  - type: aggregator
    content: "<aggregator configuration>"
  - type: finding_comparison
    content: "<finding counts and timestamps across regions>"
severity: MEDIUM
mitigation:
  immediate: "Verify aggregator configuration and allow propagation time"
  long_term: "Monitor propagation times and use EventBridge for real-time needs"
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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
