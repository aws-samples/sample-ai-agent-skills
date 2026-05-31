---
title: "H2 — Global Table Conflict Resolution"
description: "Understand last-writer-wins semantics and design patterns for conflict avoidance"
status: active
severity: MEDIUM
triggers:
  - "conflict resolution"
  - "last writer wins"
  - "concurrent writes"
  - "global table conflict"
  - "data inconsistency"
owner: devops-agent
objective: "Understand and implement conflict resolution strategies for global tables"
context: "DynamoDB Global Tables use last-writer-wins (LWW) conflict resolution based on timestamps. When the same item is written in multiple regions simultaneously, the write with the latest timestamp wins. There is no merge or conflict detection — the losing write is silently overwritten. Applications must design around this by using region-specific writes, conditional writes, or conflict-free data structures."
---

## Phase 1 — Triage

MUST:
- Identify if concurrent writes to the same item occur across regions
- Check replication latency to understand the conflict window: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ReplicationLatency --dimensions Name=TableName,Value=<table> Name=ReceivingRegion,Value=<region> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Review application architecture for multi-region write patterns
- Check if data inconsistencies have been reported

SHOULD:
- Analyze write patterns to determine if conflicts are likely
- Check if the application uses conditional writes to detect conflicts
- Review if region-affinity routing is implemented (each item written in one region)
- Check for data inconsistencies that indicate lost writes

MAY:
- Estimate the conflict window based on replication latency
- Review if CRDT (Conflict-free Replicated Data Type) patterns could be applied
- Check if the application tracks write timestamps for conflict detection

## Phase 2 — Remediate

MUST:
- Choose a conflict avoidance strategy:
  - Region affinity: route writes for each item to a single region
  - Conditional writes: use ConditionExpression with version numbers
  - Append-only: use sort key with timestamp to avoid overwrites
- Document the chosen strategy for all developers

SHOULD:
- Implement region-affinity routing: assign each item to a primary write region based on partition key
- Use atomic counters (ADD operation) instead of read-modify-write for numeric fields
- Design data models to minimize concurrent cross-region writes to the same item

MAY:
- Implement application-level conflict detection using version attributes
- Use DynamoDB Streams to detect and resolve conflicts after the fact
- Consider using a single-region write pattern with read replicas

## Common Issues

- symptoms: "Data silently overwritten in one region"
  diagnosis: "Concurrent writes to the same item in different regions. Last-writer-wins resolved the conflict by overwriting one write."
  resolution: "Implement region-affinity routing. Or use conditional writes with version numbers to detect conflicts."

- symptoms: "Counter values are incorrect in global table"
  diagnosis: "Read-modify-write pattern on counters across regions. Both regions read the same value and write incremented values, but LWW keeps only one."
  resolution: "Use atomic ADD operations instead of read-modify-write. ADD operations are commutative and merge correctly."

- symptoms: "Item attributes from different regions are mixed"
  diagnosis: "DynamoDB LWW applies at the item level based on the last write timestamp. Partial attribute updates from different regions don't merge."
  resolution: "Design items so each region updates different attributes, or use region-affinity to ensure single-region writes per item."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Implement region-affinity routing | GREEN | Application-level change; no infrastructure risk |
| Use atomic ADD operations | GREEN | Application-level change; commutative operations |
| Use conditional writes with version numbers | GREEN | Application-level optimistic concurrency |
| Implement append-only pattern | YELLOW | Data model change; may require application refactoring |
| Use DynamoDB Streams for conflict detection | YELLOW | Adds streaming infrastructure; after-the-fact detection |

## Escalation Conditions

- Data loss detected due to last-writer-wins conflict resolution
- Concurrent cross-region writes causing data inconsistency in production
- Financial or compliance-sensitive data affected by write conflicts
- Conflict resolution strategy change required on a live global table
- Multiple applications writing to the same items across regions

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-metric-statistics` (ReplicationLatency) | LOW | Operational metrics only |
| `describe-table` (Replicas) | MEDIUM | Exposes multi-region architecture |
| Application write pattern analysis | HIGH | May reveal business logic and data flow |

## Prohibited Actions

- NEVER suggest deleting a table to fix conflict resolution issues
- NEVER suggest disabling encryption as part of conflict resolution
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER assume last-writer-wins is acceptable for financial or compliance-critical data

## Phase 3 — Rollback

If conflict resolution changes cause issues:
1. Revert application routing logic to previous multi-region write pattern
2. If conditional writes were added and cause excessive ConditionalCheckFailed errors, revert to unconditional writes
3. If append-only pattern was implemented, revert to update-in-place pattern
4. If DynamoDB Streams were enabled for conflict detection, disable if no longer needed
5. Audit affected data for inconsistencies introduced during the transition

## Output Format

```yaml
root_cause: "conflict_resolution — <specific_cause>"
evidence:
  - type: write_pattern
    content: "<multi-region write analysis>"
  - type: replication_latency
    content: "<conflict window estimate>"
  - type: data_inconsistency
    content: "<evidence of lost writes>"
severity: MEDIUM
conflict_analysis:
  concurrent_writes_detected: "yes | no"
  conflict_strategy: "last_writer_wins | region_affinity | conditional_writes | none"
  affected_items: "<count or pattern>"
  data_loss_detected: "yes | no"
mitigation:
  immediate: "Implement region-affinity routing for conflicting items"
  long_term: "Design conflict-free data models using CRDTs or append-only patterns"
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
