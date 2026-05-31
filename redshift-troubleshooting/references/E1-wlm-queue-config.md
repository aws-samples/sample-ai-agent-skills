---
title: "E1 — WLM Queue Configuration"
description: "Diagnose and resolve WLM queue configuration issues"
status: active
severity: HIGH
triggers:
  - "WLM"
  - "workload management"
  - "queue configuration"
  - "query routing"
  - "memory allocation"
  - "service class"
owner: devops-agent
objective: "Optimize WLM queue configuration for workload requirements"
context: "WLM controls query concurrency, memory allocation, and query routing. Auto WLM dynamically adjusts resources. Manual WLM requires explicit configuration of queues, concurrency, and memory percentages. Misconfigured WLM causes queue waits, OOM errors, or resource waste."
---

## Phase 1 — Triage

MUST:
- Check WLM mode (Auto vs Manual): `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].ClusterParameterGroups'`
- Check WLM configuration: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT service_class, name, num_query_tasks, query_working_mem, max_execution_time, user_group, query_group FROM stv_wlm_service_class_config WHERE service_class >= 6"`
- Check queue utilization: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT service_class, num_executing_queries, num_queued_queries, avg_queue_time, avg_exec_time FROM stv_wlm_service_class_state WHERE service_class >= 6"`
- Check parameter group WLM JSON: `aws redshift describe-cluster-parameters --parameter-group-name <param-group> --query "Parameters[?ParameterName=='wlm_json_configuration'].ParameterValue"`

SHOULD:
- Check if queries are routed to correct queues
- Verify memory allocation totals 100% across all queues
- Check for queries hitting max_execution_time timeout
- Review Short Query Acceleration (SQA) configuration

MAY:
- Analyze query patterns to determine optimal queue structure
- Check query monitoring rules (QMR) configuration
- Review historical queue metrics for capacity planning

## Phase 2 — Remediate

MUST:
- If manual WLM with poor performance: switch to Auto WLM (recommended for most workloads)
- If memory allocation imbalanced: adjust memory percentages across queues
- If queries timing out: increase max_execution_time or move to appropriate queue
- If queries in wrong queue: fix user_group or query_group routing rules

SHOULD:
- Enable SQA for short queries (< 20 seconds predicted runtime)
- Create separate queues for ETL (high memory, low concurrency) and BI (lower memory, higher concurrency)
- Set query monitoring rules to log or abort expensive queries

MAY:
- Implement priority-based routing with queue hopping
- Use query_group labels in application connection strings
- Configure queue assignment rules based on user groups

## Common Issues

- symptoms: "All queries go to the default queue, custom queues are empty"
  diagnosis: "Query routing rules (user_group, query_group) not matching."
  resolution: "Set query_group before running queries: SET query_group TO 'etl'. Or configure user_group mapping."

- symptoms: "Queries fail with 'out of memory' in WLM"
  diagnosis: "Query needs more memory than allocated to its WLM queue slot."
  resolution: "Increase memory percentage for the queue, reduce concurrency, or switch to Auto WLM."

## Output Format

```yaml
root_cause: "WLM configuration — <specific_cause>"
evidence:
  - type: system_table
    content: "<stv_wlm_service_class_config output>"
severity: HIGH
mitigation:
  immediate: "Adjust WLM configuration or switch to Auto WLM"
  long_term: "Implement proper queue structure with routing rules, enable SQA, set QMR"
```


## Safety Ratings
```
safety_ratings:
  - "Query stv_wlm_service_class_config: GREEN — read-only system table query"
  - "Check WLM queue utilization: GREEN — read-only system table query"
  - "Check parameter group: GREEN — read-only API call"
  - "Switch to Auto WLM: YELLOW — parameter group change, may require cluster restart"
  - "Modify WLM queue memory allocation: YELLOW — affects all queries in the queue"
  - "Set max_execution_time: YELLOW — may abort long-running queries"
  - "Configure query monitoring rules: YELLOW — may auto-abort expensive queries"
```

## Escalation Conditions
- Cluster serves production analytics
- WLM misconfiguration causing widespread query failures or OOM errors
- Fix requires parameter group change and cluster restart
- Query routing rules affecting critical business users
- Memory allocation changes impacting ETL and BI workloads simultaneously

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "WLM configuration: resource allocation and routing rules"
    - "User groups and query groups: organizational access patterns"
    - "Query text in queue: may contain sensitive SQL"
  handling: "Do not expose WLM configuration or user group mappings externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER set total WLM memory allocation above 100% across all queues
- NEVER set concurrency to 0 on any active queue
- NEVER apply WLM changes during peak hours without a rollback plan

## Phase 3 — Rollback
- If WLM was switched to Auto: revert parameter group to previous manual WLM JSON
- If memory allocation was changed: restore previous memory percentages in parameter group
- If max_execution_time was set: remove or increase the timeout
- If query monitoring rules were added: remove the rules from the parameter group
- If SQA was enabled: disable SQA in the parameter group

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
