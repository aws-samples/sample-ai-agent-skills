---
title: "B2 — Index Management Issues"
description: "Diagnose index creation, usage, and optimization problems on DocumentDB"
status: active
severity: MEDIUM
triggers:
  - "index creation"
  - "missing index"
  - "index build"
  - "index performance"
  - "unused index"
owner: devops-agent
objective: "Identify and resolve index management issues on DocumentDB"
context: "DocumentDB indexes are critical for query performance. Index builds are foreground operations that block writes. Issues include missing indexes, unused indexes consuming storage, index build failures, and incorrect index design for query patterns."
---

## Phase 1 — Triage

MUST:
- List indexes on the collection: `mongo --tls --host <endpoint> --eval "db.collection.getIndexes()"`
- Check for collection scans in explain output for common queries
- Verify index build status: `mongo --tls --host <endpoint> --eval "db.currentOp({'msg': /index/})"`
- Check index sizes: `mongo --tls --host <endpoint> --eval "db.collection.stats().indexSizes"`

SHOULD:
- Review query patterns to identify missing indexes
- Check for duplicate or redundant indexes
- Verify compound index field order matches query patterns
- Check profiler for queries not using indexes

MAY:
- Analyze index usage statistics
- Review TTL index configuration and effectiveness
- Check for indexes on low-cardinality fields

## Phase 2 — Remediate

MUST:
- Create indexes for frequently scanned query patterns
- Plan index builds during maintenance windows (blocks writes)
- Verify new indexes are used by queries via explain()

SHOULD:
- Remove unused or redundant indexes to save storage
- Design compound indexes with equality fields first, then sort, then range
- Monitor index build progress for large collections

MAY:
- Implement index management automation
- Document index strategy for each collection

## Common Issues

- symptoms: "Index creation blocks writes for extended period"
  diagnosis: "DocumentDB index builds are foreground and block writes on the collection."
  resolution: "Schedule index creation during maintenance windows. Monitor progress with currentOp()."

- symptoms: "Query not using expected index"
  diagnosis: "Index field order does not match query pattern or query optimizer chooses a different plan."
  resolution: "Verify compound index field order matches query. Use hint() to force index selection if needed."

- symptoms: "Too many indexes slowing write performance"
  diagnosis: "Each write must update all indexes on the collection, adding overhead."
  resolution: "Remove unused indexes. Consolidate overlapping indexes into compound indexes."

## Output Format

```yaml
root_cause: "index_management — <specific_cause>"
evidence:
  - type: indexes
    content: "<collection indexes and sizes>"
  - type: explain
    content: "<query explain showing index usage>"
  - type: profiler
    content: "<slow queries related to indexing>"
severity: MEDIUM
mitigation:
  immediate: "Create missing indexes or remove unused ones"
  long_term: "Implement index strategy aligned with query patterns"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Index management is operational |
| Blast Radius | HIGH — Index builds block writes; dropping indexes degrades query performance |
| Rollback Complexity | HIGH — Dropped indexes must be rebuilt (blocks writes again) |
| Compliance Scope | LOW — Index management is operational |

## Escalation Conditions

- Escalate to DBA team before creating or dropping indexes on production
- Escalate to application teams if index changes affect query performance
- Escalate to operations if index builds block writes during business hours

## Data Sensitivity

- **Index definitions** reveal query patterns — operational data
- **Index sizes** reveal data distribution

## PROHIBITED Actions

- Do NOT create indexes on production during peak hours — they block writes
- Do NOT drop indexes without confirming they are unused
- Do NOT create indexes without verifying they match query patterns via explain()

## Phase 3 — Rollback

- If an index was dropped incorrectly, recreate it (schedule during maintenance window)
- If an index build is blocking writes, it cannot be cancelled — wait for completion
- Revert application query changes if they depended on the dropped index

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
