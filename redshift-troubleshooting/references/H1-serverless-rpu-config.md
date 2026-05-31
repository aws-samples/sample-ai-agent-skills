---
title: "H1 — Serverless RPU Configuration"
description: "Diagnose and resolve Redshift Serverless RPU and performance issues"
status: active
severity: HIGH
triggers:
  - "Serverless"
  - "RPU"
  - "base capacity"
  - "Serverless slow"
  - "workgroup"
owner: devops-agent
objective: "Optimize Redshift Serverless RPU configuration for workload performance"
context: "Redshift Serverless uses RPUs (Redshift Processing Units) for compute. Base capacity sets the minimum RPU allocation (min 8, increments of 8). Serverless auto-scales RPUs based on workload. You pay for RPU-seconds of compute time. There are no nodes to manage."
---

## Phase 1 — Triage

MUST:
- Check workgroup configuration: `aws redshift-serverless get-workgroup --workgroup-name <workgroup> --query 'workgroup.{name:workgroupName,status:status,baseRPU:baseCapacity,maxRPU:maxCapacity}'`
- Check namespace: `aws redshift-serverless get-namespace --namespace-name <namespace>`
- Check compute usage: `aws cloudwatch get-metric-statistics --namespace AWS/Redshift-Serverless --metric-name ComputeSeconds --dimensions Name=Workgroup,Value=<workgroup> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Sum`
- Check query performance: `aws redshift-data execute-statement --workgroup-name <workgroup> --database <db> --sql "SELECT query_id, elapsed_time, queue_time FROM sys_query_history ORDER BY start_time DESC LIMIT 20"`

SHOULD:
- Check if base capacity is sufficient for workload
- Monitor RPU utilization patterns over time
- Check for cold start delays (first query after idle period)

MAY:
- Compare costs between Serverless and provisioned
- Check usage limits configuration
- Review query patterns for optimization opportunities

## Phase 2 — Remediate

MUST:
- If queries slow due to low RPU: increase base capacity: `aws redshift-serverless update-workgroup --workgroup-name <workgroup> --base-capacity <rpus>`
- If workgroup not active: check namespace status and VPC configuration
- If cold start delays: increase base capacity to reduce scale-from-zero time

SHOULD:
- Set base capacity based on typical workload (not peak)
- Configure usage limits to control costs
- Use Serverless query monitoring views (sys_query_history)

MAY:
- Implement scheduled capacity changes for predictable workloads
- Set up cost alerts for RPU usage
- Consider provisioned cluster for steady-state workloads

## Common Issues

- symptoms: "First query after idle period takes 30+ seconds"
  diagnosis: "Serverless scales from zero — cold start while provisioning compute."
  resolution: "Increase base capacity. Serverless keeps compute warm when base capacity is set higher."

- symptoms: "Queries consistently slow despite high base capacity"
  diagnosis: "Query optimization issue, not RPU capacity. Check query plans and table design."
  resolution: "Analyze query plans, check distribution keys, sort keys, and VACUUM status."

## Output Format

```yaml
root_cause: "Serverless RPU — <specific_cause>"
evidence:
  - type: workgroup_config
    content: "<workgroup RPU configuration>"
severity: HIGH
mitigation:
  immediate: "Adjust base RPU capacity"
  long_term: "Monitor RPU usage patterns, set usage limits, optimize queries"
```


## Safety Ratings
```
safety_ratings:
  - "Check workgroup configuration: GREEN — read-only API call"
  - "Check namespace: GREEN — read-only API call"
  - "Check CloudWatch compute metrics: GREEN — read-only monitoring query"
  - "Increase base RPU capacity: YELLOW — increases cost, improves performance"
  - "Decrease base RPU capacity: YELLOW — reduces cost, may increase cold start time"
  - "Configure usage limits: GREEN — sets cost controls without affecting current queries"
```

## Escalation Conditions
- Cluster serves production analytics
- Cold start delays impacting time-sensitive queries
- RPU costs exceeding budget
- Base capacity insufficient for workload requirements
- Usage limits being hit causing query throttling

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "RPU configuration: capacity and cost details"
    - "Query performance metrics: workload characteristics"
    - "Usage limits: cost control settings"
  handling: "Do not expose RPU configuration or cost details externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER set base capacity to minimum (8 RPU) for production workloads without testing
- NEVER remove usage limits without cost approval
- NEVER change RPU configuration during active critical queries

## Phase 3 — Rollback
- If base capacity was increased: decrease back to previous RPU value
- If base capacity was decreased: increase back to previous RPU value if performance degrades
- If usage limits were configured: remove or adjust usage limits
- If workgroup was modified: update workgroup back to previous configuration

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
