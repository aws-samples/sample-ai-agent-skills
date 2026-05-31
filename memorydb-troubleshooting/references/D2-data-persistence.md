---
title: "D2 — Data Persistence vs ElastiCache"
description: "Clarify data persistence differences between MemoryDB and ElastiCache"
status: active
severity: MEDIUM
triggers:
  - "data persistence"
  - "memorydb vs elasticache"
  - "which service"
  - "data durability"
  - "cache vs database"
owner: devops-agent
objective: "Help users understand data persistence differences and choose the right service"
context: "MemoryDB provides durable storage with a transaction log. ElastiCache is a cache without durability guarantees. Choosing the wrong service leads to either unnecessary cost (MemoryDB for caching) or data loss risk (ElastiCache for primary data)."
---

## Phase 1 — Triage

MUST:
- Identify the use case: primary data store vs cache
- Check if data can be regenerated from another source
- Verify current service being used: `aws memorydb describe-clusters` or `aws elasticache describe-cache-clusters`
- Assess data loss impact for the workload

SHOULD:
- Compare latency requirements (sub-ms writes → ElastiCache, durability → MemoryDB)
- Review cost implications (MemoryDB is more expensive)
- Check if application handles cache misses (suggests caching use case)
- Verify backup/snapshot requirements

MAY:
- Analyze read/write ratio and patterns
- Review application architecture for data flow
- Compare feature requirements between services

## Phase 2 — Remediate

MUST:
- Use MemoryDB when data durability is required (primary database, session store, leaderboard)
- Use ElastiCache when data can be regenerated (cache layer, temporary data)
- Ensure application architecture matches the service's durability guarantees

SHOULD:
- Document the rationale for service selection
- Implement appropriate backup strategies for each service
- Monitor costs and optimize service selection

MAY:
- Plan migration between services if the wrong one was chosen
- Implement hybrid architecture (MemoryDB for durable data, ElastiCache for caching)

## Common Issues

- symptoms: "Data lost after ElastiCache node failure"
  diagnosis: "ElastiCache is a cache — data loss on failure is expected behavior."
  resolution: "If durability is needed, migrate to MemoryDB. If caching, implement cache-aside pattern."

- symptoms: "MemoryDB costs higher than expected"
  diagnosis: "MemoryDB pricing includes transaction log infrastructure for durability."
  resolution: "If data can be regenerated, consider ElastiCache for lower cost. MemoryDB cost is justified for durable data."

- symptoms: "Need to migrate from ElastiCache to MemoryDB"
  diagnosis: "Application requires durability that ElastiCache cannot provide."
  resolution: "Use MemoryDB's ability to snapshot ElastiCache clusters, or use online migration."

## Output Format

```yaml
root_cause: "data_persistence — <specific_cause>"
evidence:
  - type: use_case
    content: "<workload requirements analysis>"
  - type: service
    content: "<current service and configuration>"
  - type: comparison
    content: "<MemoryDB vs ElastiCache feature comparison>"
severity: MEDIUM
mitigation:
  immediate: "Verify correct service for the use case"
  long_term: "Document service selection criteria and review periodically"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete a MemoryDB cluster without verifying snapshot backups and data durability requirements
- **NEVER** modify ACL rules on production clusters without testing authentication changes in a staging environment
- **NEVER** force a failover on a production cluster during peak traffic without customer approval
- **NEVER** reduce the number of shards or replicas on a cluster serving production workloads without capacity analysis
- **NEVER** disable encryption in-transit or at-rest on clusters handling sensitive data

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing ACL authentication"
  - "NEVER suggest public subnet placement"
