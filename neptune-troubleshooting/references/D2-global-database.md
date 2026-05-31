---
title: "D2 — Global Database"
description: "Diagnose Neptune Global Database replication and failover issues"
status: active
severity: HIGH
triggers:
  - "global database"
  - "cross region replication"
  - "global cluster"
  - "secondary region"
owner: devops-agent
objective: "Identify and resolve Neptune Global Database issues"
context: "Neptune Global Database replicates the entire storage volume to secondary regions with typical lag under 1 second. This is the only way to achieve cross-region replication — Neptune does not support cross-region read replicas."
---

## Phase 1 — Triage

MUST:
- Describe global clusters: `aws neptune describe-global-clusters`
- Check global cluster members: `aws neptune describe-global-clusters --global-cluster-identifier <global-cluster-id>`
- Check replication lag: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name GlobalReplicationLag --dimensions Name=DBClusterIdentifier,Value=<secondary-cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Verify secondary cluster status: `aws neptune describe-db-clusters --db-cluster-identifier <secondary-cluster-id>`

SHOULD:
- Check primary cluster health and write throughput
- Monitor secondary cluster instance status
- Verify network connectivity between regions
- Check for global cluster events

MAY:
- Test failover procedure in non-production
- Verify application DNS failover configuration
- Check cross-region IAM permissions

## Phase 2 — Remediate

MUST:
- Address replication lag by investigating primary cluster write load
- Verify secondary cluster instances are healthy
- Ensure global cluster configuration is correct

SHOULD:
- Set up CloudWatch alarms for GlobalReplicationLag
- Document failover procedures
- Test failover regularly

MAY:
- Implement automated failover with Route 53 health checks
- Configure cross-region monitoring

## Common Issues

- symptoms: "High global replication lag"
  diagnosis: "Heavy write workload on primary or network latency between regions."
  resolution: "Monitor primary write throughput. Lag should recover after write burst."

- symptoms: "Cannot add secondary region"
  diagnosis: "Engine version incompatibility or quota limits."
  resolution: "Ensure primary uses engine version supporting global database. Check quotas."

- symptoms: "Failover to secondary region failed"
  diagnosis: "Secondary cluster must be promoted to standalone before becoming primary."
  resolution: "Remove secondary from global cluster, then promote. Update application endpoints."

## Output Format

```yaml
root_cause: "global_database — <specific_cause>"
evidence:
  - type: global_cluster
    content: "<global cluster configuration>"
  - type: replication_lag
    content: "<GlobalReplicationLag metrics>"
severity: HIGH
mitigation:
  immediate: "Address replication or failover issue"
  long_term: "Implement automated failover and monitoring"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — global databases span multiple regions with different compliance requirements
- Service Disruption Risk: HIGH — global database issues affect multi-region availability
- Compliance Impact: HIGH — cross-region data replication has regulatory implications

## Escalation Conditions
- Cross-region replication lag exceeds acceptable thresholds
- Global database failover fails or takes too long
- Secondary region cluster becomes unavailable
- Data inconsistency detected between primary and secondary regions
- Global database removal or detach operation fails

## Data Sensitivity
- Global cluster identifiers — INTERNAL
- Cross-region replication metrics — INTERNAL
- Regional endpoint addresses — INTERNAL
- Failover configurations — CONFIDENTIAL
- Never expose cross-region data flow details externally

## Prohibited Actions
- Do NOT remove a secondary region from a global database without data migration planning
- Do NOT perform unplanned failover to a secondary region without coordination
- Do NOT ignore cross-region replication lag warnings
- Do NOT modify global database settings during active failover
- Do NOT create global databases across regions with incompatible compliance requirements
- Do NOT delete the primary cluster without detaching secondary clusters first

## Phase 3 — Rollback
- If failover was unplanned, fail back to the original primary region when ready
- Restore secondary cluster if accidentally removed from global database
- Revert global database configuration changes
- Verify data consistency across all regions after any global database changes
- Re-establish replication monitoring after topology changes
- Document global database changes and cross-region impact

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
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"
