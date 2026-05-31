---
title: "A2 — Queue Wait Times"
description: "Diagnose and resolve high WLM queue wait times in Redshift"
status: active
severity: HIGH
triggers:
  - "queue wait"
  - "queries waiting"
  - "WLM queue"
  - "queued queries"
  - "concurrency limit"
owner: devops-agent
objective: "Identify why queries are waiting in WLM queues and reduce wait times"
context: "WLM queues control query concurrency and memory allocation. When all slots in a queue are occupied, new queries wait. High queue wait times indicate either too many concurrent queries, insufficient concurrency slots, or long-running queries blocking slots."
---

## Phase 1 — Triage

MUST:
- Check current WLM queue state: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT * FROM stv_wlm_query_state ORDER BY queue_start_time"`
- Check WLM configuration: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT service_class, num_query_tasks, query_working_mem, max_execution_time FROM stv_wlm_service_class_config WHERE service_class >= 6"`
- Check queue wait metrics: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT service_class, num_queued_queries, avg_queue_time FROM stv_wlm_service_class_state WHERE service_class >= 6"`
- Check CloudWatch WLM metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Redshift --metric-name WLMQueueWaitTime --dimensions Name=ClusterIdentifier,Value=<cluster-id> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Average`

SHOULD:
- Check if Auto WLM is enabled: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].ClusterParameterGroups'`
- Identify long-running queries holding slots: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT query, elapsed, state FROM stv_recents WHERE status = 'Running' ORDER BY elapsed DESC"`
- Check if concurrency scaling is enabled and active

MAY:
- Review query routing rules (user groups, query groups)
- Check historical queue wait patterns over the past 24 hours
- Analyze query mix (short vs long queries in same queue)

## Phase 2 — Remediate

MUST:
- If manual WLM with low concurrency: switch to Auto WLM or increase concurrency slots
- If long-running queries blocking slots: cancel or move to a separate queue with timeout
- If burst traffic: enable concurrency scaling for read queries

SHOULD:
- Configure separate WLM queues for short and long queries
- Set max_execution_time (query timeout) on queues to prevent runaway queries
- Enable Short Query Acceleration (SQA) for sub-second queries

MAY:
- Implement query priority rules based on user groups
- Schedule heavy ETL queries during off-peak hours
- Use query monitoring rules to auto-abort expensive queries

## Common Issues

- symptoms: "Queries waiting 5+ minutes in queue before execution starts"
  diagnosis: "All concurrency slots occupied by long-running queries."
  resolution: "Switch to Auto WLM. Set max_execution_time. Separate short and long query queues."

- symptoms: "Concurrency scaling not activating despite queue waits"
  diagnosis: "Concurrency scaling not enabled or only handles read queries."
  resolution: "Enable concurrency scaling on the WLM queue. Note: only read queries are offloaded."

## Output Format

```yaml
root_cause: "Queue wait — <specific_cause>"
evidence:
  - type: system_table
    content: "<stv_wlm_query_state or stv_wlm_service_class_config output>"
severity: HIGH
mitigation:
  immediate: "Adjust WLM concurrency or cancel blocking queries"
  long_term: "Switch to Auto WLM, enable concurrency scaling, separate query queues"
```


## Safety Ratings
```
safety_ratings:
  - "Query stv_wlm_query_state: GREEN — read-only system table query"
  - "Check WLM configuration: GREEN — read-only inspection"
  - "Cancel blocking query: YELLOW — terminates a running query, may affect user"
  - "Switch to Auto WLM: YELLOW — parameter group change, may require cluster restart"
  - "Enable concurrency scaling: YELLOW — adds cost for burst capacity"
  - "Modify WLM queue concurrency: YELLOW — parameter group change affects all queries in queue"
```

## Escalation Conditions
- Cluster serves production analytics
- Queue wait times exceeding SLA thresholds
- Fix requires WLM configuration change (parameter group update)
- Long-running queries from critical ETL jobs blocking BI queries
- Concurrency scaling cost implications need approval

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Query text in stv_recents: may contain sensitive SQL"
    - "User credentials: database usernames and session info"
    - "WLM configuration: resource allocation details"
  handling: "Do not expose query text or user session details externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER cancel queries without confirming they are not critical ETL or data pipeline operations
- NEVER set WLM concurrency to 0 (disables the queue)
- NEVER remove all WLM queues — at least one user queue must exist

## Phase 3 — Rollback
- If WLM was switched to Auto: revert parameter group to previous manual WLM JSON configuration
- If concurrency was increased: reduce concurrency back to previous value in parameter group
- If concurrency scaling was enabled: disable concurrency scaling on the WLM queue
- If max_execution_time was set: remove or increase the timeout value
- If a query was cancelled: user must re-submit the query

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
