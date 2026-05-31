---
title: "B3 — Profiler Issues"
description: "Diagnose DocumentDB profiler configuration and analysis problems"
status: active
severity: LOW
triggers:
  - "profiler"
  - "slow log"
  - "query logging"
  - "profiler not working"
  - "profiler threshold"
owner: devops-agent
objective: "Configure and use the DocumentDB profiler for query performance analysis"
context: "The DocumentDB profiler logs slow operations to CloudWatch Logs. It is controlled via cluster parameter group settings (profiler, profiler_threshold_ms, profiler_sampling_rate). The profiler is essential for identifying slow queries and optimization opportunities."
---

## Phase 1 — Triage

MUST:
- Check profiler parameter group settings: `aws docdb describe-db-cluster-parameters --db-cluster-parameter-group-name <param-group> --filters Name=ParameterName,Values=profiler,profiler_threshold_ms,profiler_sampling_rate`
- Verify CloudWatch Logs export is enabled on the cluster: `aws docdb describe-db-clusters --db-cluster-identifier <cluster-id>`
- Check if profiler logs are appearing in CloudWatch: `aws logs describe-log-groups --log-group-name-prefix /aws/docdb/<cluster-id>/profiler`

SHOULD:
- Verify profiler is set to "enabled" in the parameter group
- Check profiler_threshold_ms value (default 100ms)
- Verify profiler_sampling_rate (1.0 = all slow ops, 0.5 = 50% sampling)
- Check CloudWatch Logs for recent profiler entries

MAY:
- Analyze profiler output for common slow query patterns
- Adjust threshold and sampling rate based on needs
- Set up CloudWatch Logs Insights queries for profiler analysis

## Phase 2 — Remediate

MUST:
- Enable profiler in the cluster parameter group if disabled
- Enable CloudWatch Logs export for the profiler log type
- Set appropriate profiler_threshold_ms for the workload

SHOULD:
- Start with a higher threshold (500ms) and lower as needed
- Use CloudWatch Logs Insights to analyze profiler data
- Correlate profiler entries with application performance issues

MAY:
- Create CloudWatch dashboards for profiler metrics
- Set up alerts for queries exceeding specific thresholds

## Common Issues

- symptoms: "Profiler enabled but no logs in CloudWatch"
  diagnosis: "CloudWatch Logs export is not enabled on the cluster for the profiler log type."
  resolution: "Modify the cluster to enable CloudWatch Logs export: --enable-cloudwatch-logs-exports profiler."

- symptoms: "Too many profiler entries overwhelming CloudWatch"
  diagnosis: "Profiler threshold is too low or sampling rate is 1.0 capturing all slow operations."
  resolution: "Increase profiler_threshold_ms or reduce profiler_sampling_rate."

- symptoms: "Profiler not capturing expected slow queries"
  diagnosis: "Profiler threshold is higher than the query execution time."
  resolution: "Lower profiler_threshold_ms to capture the target queries."

## Output Format

```yaml
root_cause: "profiler — <specific_cause>"
evidence:
  - type: parameter_group
    content: "<profiler parameter settings>"
  - type: cloudwatch_logs
    content: "<profiler log entries>"
  - type: cluster_config
    content: "<CloudWatch Logs export configuration>"
severity: LOW
mitigation:
  immediate: "Enable profiler and CloudWatch Logs export"
  long_term: "Implement profiler-based query performance monitoring"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Profiler output may contain query parameters with sensitive data |
| Blast Radius | LOW — Profiler is read-only logging; does not affect database operations |
| Rollback Complexity | LOW — Profiler settings can be changed in the parameter group |
| Compliance Scope | LOW — Profiler is a diagnostic tool |

## Escalation Conditions

- Escalate to security if profiler logs contain sensitive query parameters
- Escalate to operations if profiler logging volume overwhelms CloudWatch

## Data Sensitivity

- **Profiler logs** may contain query parameters with sensitive data — restrict CloudWatch access
- **Query patterns** revealed by profiler may be operationally sensitive

## PROHIBITED Actions

- Do NOT set profiler_threshold_ms too low on high-traffic clusters — it generates excessive logs
- Do NOT share profiler logs containing sensitive query parameters
- Do NOT leave profiler at 100% sampling rate on production indefinitely

## Phase 3 — Rollback

- Restore previous profiler settings in the parameter group
- Adjust profiler_threshold_ms and profiler_sampling_rate to previous values
- Delete excessive profiler logs from CloudWatch if needed

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
  - "NEVER suggest disabling TLS"
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
