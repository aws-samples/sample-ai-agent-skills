---
title: "E2 — Concurrency Scaling"
description: "Diagnose and resolve concurrency scaling issues"
status: active
severity: MEDIUM
triggers:
  - "concurrency scaling"
  - "burst capacity"
  - "scaling cluster"
  - "transient cluster"
owner: devops-agent
objective: "Ensure concurrency scaling activates properly for burst read workloads"
context: "Concurrency scaling adds transient clusters to handle burst read query workloads. It only handles read queries (SELECT). Write operations always run on the main cluster. Each cluster gets 1 hour of free concurrency scaling credits per day."
---

## Phase 1 — Triage

MUST:
- Check concurrency scaling usage: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT * FROM stl_concurrency_scaling_usage ORDER BY starttime DESC LIMIT 20"`
- Check if concurrency scaling is enabled on WLM queues: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT service_class, name, concurrency_scaling FROM stv_wlm_service_class_config WHERE service_class >= 6"`
- Check CloudWatch concurrency scaling metric: `aws cloudwatch get-metric-statistics --namespace AWS/Redshift --metric-name ConcurrencyScalingActiveClusters --dimensions Name=ClusterIdentifier,Value=<cluster-id> --start-time $(date -u -d '6 hours ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Maximum`
- Verify cluster parameter: `aws redshift describe-cluster-parameters --parameter-group-name <param-group> --query "Parameters[?ParameterName=='max_concurrency_scaling_clusters'].ParameterValue"`

SHOULD:
- Check if write queries are being sent to concurrency scaling (they won't be offloaded)
- Verify concurrency scaling credits usage
- Check queue wait times to confirm scaling is needed

MAY:
- Review billing for concurrency scaling usage
- Check if max_concurrency_scaling_clusters limit is too low
- Analyze query patterns to determine if scaling is cost-effective

## Phase 2 — Remediate

MUST:
- If not enabled: enable concurrency scaling on WLM queue: set `concurrency_scaling` mode to `auto`
- If not activating: check max_concurrency_scaling_clusters parameter (default 1)
- If write queries not offloaded: this is expected — concurrency scaling is read-only

SHOULD:
- Increase max_concurrency_scaling_clusters for heavy burst workloads
- Monitor concurrency scaling costs vs permanent node addition
- Use concurrency scaling credits (1 hour free per day per cluster)

MAY:
- Consider Serverless for unpredictable workloads instead of concurrency scaling
- Implement query routing to separate read and write workloads
- Set up cost alerts for concurrency scaling usage

## Common Issues

- symptoms: "Concurrency scaling enabled but never activates"
  diagnosis: "Queue never reaches full concurrency — no waiting queries to trigger scaling."
  resolution: "Concurrency scaling only activates when queries wait in queue. If no waits, scaling is not needed."

- symptoms: "Write queries still slow despite concurrency scaling"
  diagnosis: "Concurrency scaling only handles read queries (SELECT)."
  resolution: "Optimize write queries directly. Concurrency scaling cannot offload INSERT, UPDATE, DELETE, or COPY."

## Output Format

```yaml
root_cause: "Concurrency scaling — <specific_cause>"
evidence:
  - type: system_table
    content: "<stl_concurrency_scaling_usage output>"
severity: MEDIUM
mitigation:
  immediate: "Enable concurrency scaling or adjust max clusters"
  long_term: "Monitor scaling costs, consider Serverless for unpredictable workloads"
```


## Safety Ratings
```
safety_ratings:
  - "Query stl_concurrency_scaling_usage: GREEN — read-only system table query"
  - "Check WLM queue concurrency scaling setting: GREEN — read-only inspection"
  - "Check CloudWatch scaling metrics: GREEN — read-only monitoring query"
  - "Enable concurrency scaling on queue: YELLOW — adds cost for burst capacity"
  - "Increase max_concurrency_scaling_clusters: YELLOW — increases potential cost"
```

## Escalation Conditions
- Cluster serves production analytics
- Concurrency scaling costs exceeding budget
- Scaling not activating despite queue waits
- Write query performance not improved (scaling is read-only)
- Cost approval needed for increased scaling limits

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Scaling usage metrics: cost and capacity data"
    - "Query patterns: workload characteristics"
  handling: "Do not expose scaling cost details or query patterns externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER set max_concurrency_scaling_clusters to an unlimited value without cost controls
- NEVER expect concurrency scaling to handle write queries (INSERT, UPDATE, DELETE, COPY)

## Phase 3 — Rollback
- If concurrency scaling was enabled: disable it on the WLM queue to stop additional costs
- If max_concurrency_scaling_clusters was increased: reduce back to previous value
- If scaling is causing unexpected costs: disable immediately and investigate query patterns

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
