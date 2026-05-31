---
title: "H3 — Region Failover Strategy"
description: "Implement failover strategies for global tables using Route53 and application-level routing"
status: active
severity: MEDIUM
triggers:
  - "region failover"
  - "disaster recovery"
  - "multi-region failover"
  - "Route53 failover"
  - "region outage"
owner: devops-agent
objective: "Design and implement reliable region failover for DynamoDB global tables"
context: "DynamoDB Global Tables replicate data across regions but do not provide automatic application failover. Applications must implement their own failover logic using Route53 health checks, application-level routing, or custom health monitoring. Failover should consider replication lag to avoid reading stale data. Global Tables Version 2019.11.21 supports active-active multi-region writes."
---

## Phase 1 — Triage

MUST:
- Check replica status in all regions: `aws dynamodb describe-table --table-name <table> --query 'Table.Replicas'`
- Check replication latency: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ReplicationLatency --dimensions Name=TableName,Value=<table> Name=ReceivingRegion,Value=<region> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Verify the application has failover logic implemented
- Check Route53 health check status if configured: `aws route53 list-health-checks`

SHOULD:
- Check if the application can detect region failures (health checks, error rates)
- Verify DNS TTL settings for Route53 failover records
- Check if the application handles failover gracefully (connection draining, retry logic)
- Review if replication lag is acceptable for failover reads

MAY:
- Test failover by simulating a region failure
- Check if the application uses multi-region endpoints
- Review RTO (Recovery Time Objective) and RPO (Recovery Point Objective) requirements

## Phase 2 — Remediate

MUST:
- Implement health checks for the DynamoDB endpoint in each region
- Configure application-level failover logic to switch regions on failure
- Ensure the application can write to any replica region (active-active)

SHOULD:
- Use Route53 health checks with failover routing: `aws route53 create-health-check --caller-reference <ref> --health-check-config Type=HTTPS,FullyQualifiedDomainName=<api-endpoint>,Port=443,ResourcePath=/health`
- Set DNS TTL low (60 seconds) for faster failover
- Implement circuit breaker pattern in the application for DynamoDB calls

MAY:
- Use AWS Global Accelerator for automatic traffic routing
- Implement custom health monitoring that checks DynamoDB latency and error rates
- Create runbooks for manual failover procedures

## Common Issues

- symptoms: "Application fails when primary region goes down"
  diagnosis: "No failover logic implemented. Application only connects to one region."
  resolution: "Implement multi-region connection logic. Use Route53 failover routing or application-level region switching."

- symptoms: "Failover works but users see stale data"
  diagnosis: "Replication lag means the failover region doesn't have the latest writes."
  resolution: "Accept RPO based on replication latency (typically < 1 second). For critical data, implement read-after-write consistency checks."

- symptoms: "Failover takes too long"
  diagnosis: "DNS TTL is too high, or health check intervals are too long."
  resolution: "Reduce DNS TTL to 60 seconds. Set health check interval to 10 seconds with 2 failure threshold."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Implement health checks | GREEN | Monitoring only; no operational impact |
| Configure Route53 failover routing | YELLOW | DNS change; affects traffic routing for all clients |
| Set DNS TTL to 60 seconds | YELLOW | DNS change; increases DNS query volume |
| Implement circuit breaker pattern | GREEN | Application-level resilience; no infrastructure risk |
| Execute manual failover | RED | Switches primary region; brief write unavailability; potential data loss |

## Escalation Conditions

- Primary region experiencing outage requiring failover decision
- Replication lag too high for acceptable RPO before failover
- Application unable to connect after failover (endpoint not updated)
- Failover required but no failover procedure documented
- RTO/RPO requirements not met by current failover configuration

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` (Replicas) | MEDIUM | Exposes multi-region architecture |
| `get-metric-statistics` (ReplicationLatency) | LOW | Operational metrics only |
| `list-health-checks` (Route53) | LOW | Health check configuration only |

## Prohibited Actions

- NEVER suggest deleting a table to fix failover issues
- NEVER suggest disabling encryption on any replica region
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER execute a region failover without confirming replication lag and data loss implications

## Phase 3 — Rollback

If failover or failover configuration changes cause issues:
1. If Route53 failover routing was configured incorrectly, revert DNS records to point to the correct region
2. If DNS TTL was changed, revert to previous TTL value
3. If a failover was executed and the original primary recovers, it automatically becomes a secondary — plan failback carefully
4. If health checks are generating false positives, adjust thresholds or temporarily disable
5. Update application connection strings if they were changed during failover

## Output Format

```yaml
root_cause: "region_failover — <specific_cause>"
evidence:
  - type: replica_status
    content: "<replica health in all regions>"
  - type: health_checks
    content: "<Route53 health check status>"
  - type: failover_config
    content: "<failover routing configuration>"
severity: MEDIUM
failover_analysis:
  primary_region: "<region>"
  replica_regions: "<list>"
  failover_mechanism: "Route53 | application_level | none"
  health_check_configured: "yes | no"
  rto_seconds: "<estimated>"
  rpo_seconds: "<estimated based on replication lag>"
mitigation:
  immediate: "Implement basic failover routing"
  long_term: "Automate failover with health checks and low DNS TTL"
```

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
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting a table to fix issues"
  - "NEVER run full table scans on production tables"
