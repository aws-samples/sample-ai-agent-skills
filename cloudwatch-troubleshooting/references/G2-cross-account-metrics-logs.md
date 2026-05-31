---
title: "G2 — Cross-Account Metrics and Logs"
description: "Diagnose cross-account metric and log visibility issues"
status: active
severity: MEDIUM
triggers:
  - "cross-account metrics"
  - "cross-account logs"
  - "multi-account dashboard"
  - "cross-account alarm"
  - "shared metrics"
owner: devops-agent
objective: "Identify why metrics or logs from source accounts are not visible in the monitoring account"
context: "After OAM is configured, cross-account metrics appear with the source account ID as a dimension. Cross-account log groups are accessible via the monitoring account's console and API. Cross-account alarms and dashboards can reference source account metrics. Issues arise from incomplete OAM setup, missing resource types, or IAM permission gaps."
---

## Phase 1 — Triage

MUST:
- Verify OAM link is active in source account: `aws oam list-links`
- Verify OAM sink exists in monitoring account: `aws oam list-sinks`
- Check link includes the correct resource types
- In monitoring account, try listing cross-account metrics: `aws cloudwatch list-metrics --include-linked-accounts`

SHOULD:
- Check if the specific metric/log group is being shared (resource type must match)
- Verify the monitoring account console is set to cross-account view
- Check IAM permissions in the monitoring account for cross-account access

MAY:
- Test with a simple metric query to isolate the issue
- Check if the source account has recently created the link (propagation delay)

## Phase 2 — Remediate

MUST:
- Ensure OAM link includes the needed resource types
- Verify sink policy allows the source account
- Use `--include-linked-accounts` flag when querying cross-account metrics

SHOULD:
- Wait up to 5 minutes for new links to propagate
- Use the CloudWatch console cross-account switcher for easier navigation
- Create cross-account dashboards using source account metric references

MAY:
- Set up cross-account alarms for critical source account metrics
- Document cross-account metric naming conventions

## Common Issues

- symptoms: "Cross-account metrics not appearing in list-metrics"
  diagnosis: "Missing --include-linked-accounts flag or OAM link not active."
  resolution: "Use --include-linked-accounts. Verify OAM link status in source account."

- symptoms: "Cross-account dashboard shows no data"
  diagnosis: "Dashboard widget references incorrect account ID or metric dimensions."
  resolution: "Use the correct source account ID in metric references. Verify OAM link."

- symptoms: "Cross-account log group not accessible"
  diagnosis: "OAM link does not include AWS::Logs::LogGroup resource type."
  resolution: "Update the OAM link to include AWS::Logs::LogGroup."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Update OAM link resource types | GREEN | Additive configuration |
| Verify sink policy | GREEN | Read-only check |
| Use --include-linked-accounts flag | GREEN | Query parameter, no state change |
| Create cross-account dashboards | GREEN | New resource creation |

## Escalation Conditions

- Cross-account metrics not visible for production monitoring — escalate to operations team
- OAM link configuration changes — escalate to source account owner
- Cross-account alarm creation — escalate to alarm owner and source account owner

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws oam list-links` | MEDIUM | Exposes cross-account sharing configuration |
| `aws oam list-sinks` | MEDIUM | Exposes sink configuration |
| `aws cloudwatch list-metrics` (linked accounts) | MEDIUM | Exposes cross-account metric names |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** create cross-account alarms without coordinating with the source account owner

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Updated OAM link resource types | Revert resource types |
| Created cross-account dashboards | Delete dashboards: `aws cloudwatch delete-dashboards` |
| Created cross-account alarms | Delete alarms: `aws cloudwatch delete-alarms` |

## Output Format

```yaml
root_cause: "cross_account_data — <specific_cause>"
evidence:
  - type: oam_config
    content: "<sink and link configuration>"
  - type: metric_query
    content: "<cross-account metric query results>"
severity: MEDIUM
mitigation:
  immediate: "Fix OAM link resource types or use correct query flags"
  long_term: "Automate cross-account dashboard creation"
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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
