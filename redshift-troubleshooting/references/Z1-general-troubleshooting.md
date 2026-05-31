---
title: "Z1 — General Redshift Troubleshooting (Catch-All)"
description: "Fallback SOP for Redshift issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Redshift issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any specific runbook. It provides a broad investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check cluster/workgroup status: `aws redshift describe-clusters --cluster-identifier <cluster-id>` or `aws redshift-serverless get-workgroup --workgroup-name <workgroup>`
- Check CloudWatch key metrics:
  ```
  # CPU, Disk, Health
  aws cloudwatch get-metric-statistics --namespace AWS/Redshift \
    --metric-name CPUUtilization --dimensions Name=ClusterIdentifier,Value=<cluster-id> \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 --statistics Average
  ```
- Check recent events: `aws redshift describe-events --source-identifier <cluster-id> --source-type cluster --duration 1440`
- Check recent query errors: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT query, substring, elapsed FROM svl_qlog WHERE aborted = 1 ORDER BY starttime DESC LIMIT 20"`
- Check CloudTrail for Redshift API events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=redshift.amazonaws.com --max-results 20`

SHOULD:
- Check WLM queue state for waiting queries
- Check disk space and table health via svv_table_info
- Check for lock contention via svv_transactions
- Test connectivity to cluster endpoint

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Slow queries or high CPU → Query Performance (A1-A4)
- Disk full or high empty/unsorted → Storage (B1-B3)
- COPY/UNLOAD/Spectrum errors → Data Loading (C1-C3)
- Node failure or resize/maintenance → Cluster Operations (D1-D3)
- Queue waits or concurrency issues → WLM (E1-E2)
- Access denied or connectivity → Security (F1-F2)
- Data sharing failures → Data Sharing (G1-G2)
- Serverless RPU or endpoint → Serverless (H1-H2)
- Snapshot/restore failures → Snapshots (I1-I2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Review all system table categories systematically
- Check CloudTrail for recent configuration changes
- Verify IAM and VPC configuration
- Check for recent cluster modifications

SHOULD:
- Compare with known-good baseline configuration
- Check AWS Health Dashboard for Redshift service events
- Review parameter group settings

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<query|storage|loading|cluster|wlm|security|sharing|serverless|snapshots|unknown>"
investigation_path: "cluster status → CloudWatch metrics → system tables → <domain>"
evidence:
  - type: cluster_status
    content: "<cluster configuration summary>"
  - type: cloudwatch_metrics
    content: "<key metrics summary>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings
```
safety_ratings:
  - "Check cluster status: GREEN — read-only API call"
  - "Check CloudWatch metrics: GREEN — read-only monitoring query"
  - "Check recent events: GREEN — read-only API call"
  - "Query system tables: GREEN — read-only diagnostic queries"
  - "Check CloudTrail events: GREEN — read-only audit log query"
  - "Route to specific runbook: GREEN — diagnostic classification, no state change"
```

## Escalation Conditions
- Cluster serves production analytics
- Issue cannot be classified into a specific failure domain
- Multiple failure domains affected simultaneously
- AWS service health issue suspected
- Fix requires resize or restore

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Query results from system tables: operational data"
    - "CloudTrail events: API call history with parameters"
    - "CloudWatch metrics: operational patterns"
    - "User credentials: database usernames in query logs"
  handling: "System table queries may expose sensitive operational data. Mask usernames and query text in shared reports."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER make configuration changes without first classifying the failure domain
- NEVER apply fixes from multiple runbooks simultaneously without understanding interactions

## Phase 3 — Rollback
- For general investigation: no rollback needed — all triage steps are read-only
- If routed to a specific runbook: follow that runbook's Phase 3 rollback procedures
- If configuration changes were made during investigation: revert each change individually
- If cluster was modified: restore from snapshot as a last resort

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
