---
title: "F2 — Outpost Failover Issues"
description: "Diagnose failover behavior between Outpost and Region workloads"
status: active
severity: HIGH
triggers:
  - "outpost failover"
  - "disaster recovery outpost"
  - "outpost DR"
  - "workload migration"
owner: devops-agent
objective: "Ensure proper failover behavior between Outpost and Region during outages"
context: "Outpost failover to the Region requires pre-planned architecture. There is no automatic failover mechanism. Applications must be designed for hybrid operation with health checks and routing policies."
---

## Phase 1 — Triage

MUST:
- Check Outpost connectivity: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Verify Route 53 health checks if using DNS failover: `aws route53 list-health-checks`
- Check health check status: `aws route53 get-health-check-status --health-check-id <check-id>`
- Verify Region-side resources are available for failover

SHOULD:
- Check load balancer health if using ALB/NLB for failover
- Verify data replication status between Outpost and Region
- Review failover runbook and test results

MAY:
- Check Route 53 failover routing policies
- Review application-level health check endpoints

## Phase 2 — Remediate

MUST:
- If failover not triggering: verify health check configuration and thresholds
- If Region resources not ready: launch standby instances or verify auto-scaling
- Ensure data is synchronized between Outpost and Region before failover

SHOULD:
- Implement Route 53 failover routing with health checks
- Use multi-AZ or multi-Region architectures for critical workloads
- Regularly test failover procedures

MAY:
- Implement automated failover using EventBridge and Lambda
- Set up cross-Region data replication for stateful workloads

## Common Issues

- symptoms: "Failover to Region not happening during Outpost outage"
  diagnosis: "No automated failover mechanism configured — requires explicit setup."
  resolution: "Implement Route 53 failover routing with health checks on Outpost endpoints."

- symptoms: "Data inconsistency after failover"
  diagnosis: "Data replication lag between Outpost and Region."
  resolution: "Implement synchronous or near-synchronous replication. Accept RPO based on replication lag."

## Output Format

```yaml
root_cause: "failover — <specific_cause>"
evidence:
  - type: connectivity_status
    content: "<Outpost connectivity>"
  - type: health_checks
    content: "<Route 53 health check status>"
severity: HIGH
mitigation:
  immediate: "Execute manual failover or fix automated failover"
  long_term: "Implement and regularly test automated failover procedures"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — failover may involve data replication and migration
- Service Disruption Risk: HIGH — failover operations affect workload availability
- Compliance Impact: HIGH — failover must maintain data residency and compliance requirements

## Escalation Conditions
- Automated failover does not trigger when expected
- Failover causes data inconsistency
- Failover takes longer than RTO requirements
- Failback after failover fails
- Failover affects workloads that should not be impacted

## Data Sensitivity
- Failover configurations — INTERNAL
- Data replication status — INTERNAL
- RTO/RPO metrics — INTERNAL
- Failover runbooks — INTERNAL
- Never expose failover procedures that reveal infrastructure weaknesses

## Prohibited Actions
- Do NOT trigger manual failover without proper authorization
- Do NOT bypass failover validation procedures
- Do NOT ignore failover test results
- Do NOT modify failover configurations during active incidents
- Do NOT skip data consistency verification after failover
- Do NOT failover without confirming target capacity availability

## Phase 3 — Rollback
- Initiate failback to original configuration when ready
- Verify data consistency after failback
- Restore failover configurations to pre-incident state
- Re-enable failover monitoring and alerting
- Verify all workloads are healthy after failback
- Document failover event and update DR procedures

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
