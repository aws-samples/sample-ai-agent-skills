---
title: "D1 — Writer Failover"
description: "Diagnose Aurora writer failover events and issues"
status: active
severity: CRITICAL
triggers:
  - "failover"
  - "writer failover"
  - "writer promotion"
  - "writer unavailable"
  - "failover event"
  - "unexpected promotion"
owner: devops-agent
objective: "Identify the cause of Aurora writer failover and minimize impact"
context: "Aurora writer failover promotes a reader to writer. Typical failover time is ~30 seconds when a reader exists. The cluster endpoint DNS TTL is 5 seconds. Failover can be triggered by instance failure, AZ failure, manual failover, or maintenance. Applications must handle reconnection."
---

## Phase 1 — Triage

MUST:
- Check failover events:
  ```
  aws rds describe-events --source-identifier <cluster-id> --source-type db-cluster --duration 1440 \
    --event-categories failover
  ```
- Identify current writer:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].DBClusterMembers[?IsClusterWriter==`true`]'
  ```
- Check instance events for all cluster members:
  ```
  aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440
  ```
- Check failover priority settings:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].DBClusterMembers[].{Id:DBInstanceIdentifier,Writer:IsClusterWriter,Priority:PromotionTier}'
  ```

SHOULD:
- Check CloudWatch for instance availability gaps:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization \
    --dimensions Name=DBInstanceIdentifier,Value=<old-writer-id> \
    --start-time <before-failover> --end-time <after-failover> --period 60 --statistics Average
  ```
- Check for maintenance events: `aws rds describe-pending-maintenance-actions --resource-identifier <cluster-arn>`
- Check CloudTrail for manual failover: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=FailoverDBCluster`

MAY:
- Check Enhanced Monitoring for OS-level issues before failover
- Review application logs for connection errors during failover window

## Phase 2 — Remediate

MUST:
- Verify the new writer is healthy and accepting connections
- Verify the cluster endpoint resolves to the new writer: `nslookup <cluster-endpoint>`
- Ensure applications reconnected to the new writer

SHOULD:
- Configure failover priority tiers to control which reader gets promoted
- Ensure at least one reader exists in a different AZ for fast failover
- Implement connection retry logic with exponential backoff in applications
- Use RDS Proxy for faster failover routing (Proxy detects failover and redirects)

MAY:
- Set up CloudWatch alarms for failover events
- Implement application-level health checks against the cluster endpoint
- Test failover regularly: `aws rds failover-db-cluster --db-cluster-identifier <cluster-id>`

## Common Issues

- symptoms: "Application downtime exceeded 30 seconds during failover"
  diagnosis: "DNS caching in application or JVM. Application not retrying connections."
  resolution: "Disable DNS caching (JVM: networkaddress.cache.ttl=0). Implement retry logic. Use RDS Proxy."

- symptoms: "Wrong reader was promoted to writer"
  diagnosis: "Failover priority tiers not configured. Default promotion is arbitrary."
  resolution: "Set promotion tiers: aws rds modify-db-instance --promotion-tier 0 for preferred writer."

- symptoms: "Failover triggered unexpectedly"
  diagnosis: "Instance failure, AZ issue, or maintenance event."
  resolution: "Check events and CloudTrail. Review maintenance windows. Check instance health."

## Safety Ratings
- GREEN: describe-events, describe-db-clusters, describe-db-instances, CloudWatch CPUUtilization metrics, nslookup, CloudTrail lookup — read-only inspection
- YELLOW: modify-db-instance (promotion-tier), modify-db-cluster — recoverable configuration changes
- RED: failover-db-cluster, delete-db-cluster, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Failover caused application downtime exceeding SLA"
- "Fix requires another failover of Aurora cluster"
- "Wrong reader was promoted to writer"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, CloudTrail events (contain API caller identity and parameters)
- HIGH: failover event details (reveal infrastructure topology and failure patterns)
- MEDIUM: CloudWatch metrics, instance status, promotion tier configuration

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix failover issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest triggering another failover immediately after a failover — wait for DNS propagation and stabilization"

## Phase 3 — Rollback
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"
- "Restore from snapshot if failover resulted in data inconsistency"
- "Revert promotion tier changes if they cause undesired failover behavior"
- "Allow at least 5 minutes of stabilization before considering another failover"

## Output Format

```yaml
root_cause: "writer_failover — <specific_cause>"
evidence:
  - type: rds_events
    content: "<failover events>"
  - type: cluster_members
    content: "<current writer and reader status>"
  - type: cloudtrail
    content: "<manual failover API call if applicable>"
severity: CRITICAL
mitigation:
  immediate: "Verify new writer health and application connectivity"
  long_term: "Configure failover priorities, implement retry logic, use RDS Proxy"
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
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling encryption"
  - "NEVER force failover without understanding impact"
