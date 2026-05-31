---
title: "D2 — Global Cluster Issues"
description: "Diagnose DocumentDB global cluster replication and failover problems"
status: active
severity: HIGH
triggers:
  - "global cluster"
  - "cross-region replication"
  - "global failover"
  - "secondary region"
  - "global cluster lag"
owner: devops-agent
objective: "Identify and resolve global cluster replication and failover issues"
context: "DocumentDB global clusters provide cross-region replication with typical lag under 1 second. Secondary clusters are read-only. Failover to a secondary region is manual. Issues include replication lag, failover procedures, and secondary cluster configuration."
---

## Phase 1 — Triage

MUST:
- Describe global clusters: `aws docdb describe-global-clusters`
- Check global cluster status and member clusters
- Check replication lag to secondary regions
- Verify secondary cluster health: `aws docdb describe-db-clusters --db-cluster-identifier <secondary-cluster-id>`

SHOULD:
- Monitor cross-region replication lag via CloudWatch
- Check network connectivity between regions
- Verify secondary cluster instance health
- Review global cluster events

MAY:
- Test failover procedures in a non-production environment
- Check for region-specific service issues
- Verify KMS key configuration in secondary regions

## Phase 2 — Remediate

MUST:
- Address replication lag causes (writer overload, network issues)
- Ensure secondary clusters have healthy instances
- Document and test failover procedures

SHOULD:
- Monitor global cluster replication lag with CloudWatch alarms
- Maintain similar instance configurations across regions
- Practice failover procedures regularly

MAY:
- Automate failover decision-making with health checks
- Set up cross-region monitoring dashboards

## Common Issues

- symptoms: "High replication lag to secondary region"
  diagnosis: "Heavy write load on primary or network latency between regions."
  resolution: "Optimize write patterns on primary. Verify no network issues between regions."

- symptoms: "Cannot promote secondary cluster"
  diagnosis: "Global cluster must be detached before secondary can become standalone."
  resolution: "Use remove-from-global-cluster to detach the secondary, then it becomes a standalone read-write cluster."

- symptoms: "Secondary cluster instances not available"
  diagnosis: "Instance class not available in the secondary region or subnet group misconfigured."
  resolution: "Verify instance class availability in the secondary region. Check subnet group configuration."

## Output Format

```yaml
root_cause: "global_cluster — <specific_cause>"
evidence:
  - type: global_cluster
    content: "<global cluster status and members>"
  - type: replication
    content: "<cross-region replication lag>"
  - type: secondary
    content: "<secondary cluster health>"
severity: HIGH
mitigation:
  immediate: "Address replication lag or secondary cluster issues"
  long_term: "Implement global cluster monitoring and failover testing"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — Global clusters replicate production data across regions |
| Blast Radius | CRITICAL — Failover changes the write endpoint for all applications globally |
| Rollback Complexity | HIGH — Failover is a major operational event requiring application reconfiguration |
| Compliance Scope | CRITICAL — Cross-region replication must comply with data residency requirements |

## Escalation Conditions

- Escalate to management before performing global cluster failover
- Escalate to all application teams before failover — endpoint changes affect everyone
- Escalate to compliance if cross-region replication involves data residency concerns

## Data Sensitivity

- **Cross-region replicated data** must comply with data residency regulations
- **Global cluster configuration** reveals DR architecture
- **KMS keys** in secondary regions must be properly configured

## PROHIBITED Actions

- Do NOT perform global cluster failover without management approval
- Do NOT detach secondary clusters without understanding the impact
- Do NOT replicate data to regions that violate data residency requirements
- Do NOT ignore replication lag — it indicates potential data loss during failover

## Phase 3 — Rollback

- If failover was performed incorrectly, failover back to the original primary region
- Note: failover back requires the original region to be healthy
- Update all application connection strings to point to the correct primary
- Verify replication is healthy in both directions after rollback

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
