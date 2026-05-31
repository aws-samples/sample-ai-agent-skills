---
title: "D3 — Maintenance Issues"
description: "Diagnose and resolve Redshift maintenance window disruptions"
status: active
severity: MEDIUM
triggers:
  - "maintenance"
  - "maintenance window"
  - "cluster restart"
  - "patching"
  - "deferred maintenance"
owner: devops-agent
objective: "Manage maintenance windows to minimize disruption"
context: "Redshift applies patches during the configured maintenance window. This may restart the cluster, causing brief unavailability. Deferred maintenance allows postponing non-critical patches up to 45 days. Mandatory security patches cannot be deferred."
---

## Phase 1 — Triage

MUST:
- Check maintenance window: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].PreferredMaintenanceWindow'`
- Check pending maintenance: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].{pending:PendingModifiedValues,deferMaint:DeferMaintenanceEndTime}'`
- Check recent maintenance events: `aws redshift describe-events --source-identifier <cluster-id> --source-type cluster --duration 10080 --query "Events[?contains(Message, 'maintenance')]"`
- Check deferred maintenance status: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].DeferredMaintenanceWindows'`

SHOULD:
- Verify maintenance window is during low-traffic period
- Check if maintenance caused compilation cache clear
- Review cluster version after maintenance

MAY:
- Check AWS Health Dashboard for upcoming maintenance
- Review maintenance track (Current vs Trailing)
- Check if maintenance affected snapshot schedule

## Phase 2 — Remediate

MUST:
- If maintenance during peak hours: modify maintenance window: `aws redshift modify-cluster --cluster-identifier <cluster-id> --preferred-maintenance-window <window>`
- If non-critical patch: defer maintenance: `aws redshift modify-cluster-maintenance --cluster-identifier <cluster-id> --defer-maintenance --defer-maintenance-duration 14`
- If mandatory patch: schedule during lowest traffic period

SHOULD:
- Use Trailing maintenance track for more stable versions
- Set up notifications for upcoming maintenance
- Warm up compilation cache after maintenance restarts

MAY:
- Implement automated cache warm-up scripts
- Use multi-cluster architecture for zero-downtime maintenance
- Monitor query performance after maintenance for regressions

## Common Issues

- symptoms: "Cluster unavailable during business hours due to maintenance"
  diagnosis: "Maintenance window overlaps with peak usage."
  resolution: "Change maintenance window to off-peak hours. Defer non-critical patches."

- symptoms: "Queries slow after maintenance restart"
  diagnosis: "Compilation cache cleared by restart. First executions recompile."
  resolution: "Expected behavior. Run key queries to warm cache. Performance normalizes within minutes."

## Output Format

```yaml
root_cause: "Maintenance issue — <specific_cause>"
evidence:
  - type: cluster_config
    content: "<maintenance window and pending maintenance details>"
severity: MEDIUM
mitigation:
  immediate: "Defer maintenance or adjust maintenance window"
  long_term: "Configure optimal maintenance window, use Trailing track, implement cache warm-up"
```


## Safety Ratings
```
safety_ratings:
  - "Check maintenance window: GREEN — read-only API call"
  - "Check pending maintenance: GREEN — read-only API call"
  - "Check deferred maintenance: GREEN — read-only API call"
  - "Modify maintenance window: YELLOW — changes when patches are applied"
  - "Defer maintenance: YELLOW — postpones patches, may delay security fixes"
  - "Run cache warm-up queries: GREEN — read-only queries after restart"
```

## Escalation Conditions
- Cluster serves production analytics
- Mandatory security patch cannot be deferred
- Maintenance window overlaps with critical business operations
- Post-maintenance performance regression detected
- Compilation cache clear impacting time-sensitive dashboards

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Cluster version and patch level: infrastructure details"
    - "Maintenance schedule: operational timing"
  handling: "Do not expose cluster version or maintenance schedule externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER defer mandatory security patches beyond the maximum allowed period
- NEVER schedule maintenance during peak business hours
- NEVER skip post-maintenance validation of query performance

## Phase 3 — Rollback
- If maintenance window was changed: revert to previous maintenance window setting
- If maintenance was deferred: cancel deferral to allow patch to apply in next window
- If maintenance track was changed (Current to Trailing): change back to previous track
- Post-maintenance performance issues: warm up compilation cache, run ANALYZE on key tables

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
  - "NEVER suggest disabling encryption"
