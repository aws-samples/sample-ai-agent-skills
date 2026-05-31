---
title: "J2 — DAX Consistency Model"
description: "Understand DAX eventually consistent reads and strongly consistent bypass"
status: active
severity: MEDIUM
triggers:
  - "DAX consistency"
  - "strongly consistent DAX"
  - "DAX stale reads"
  - "DAX eventual consistency"
  - "ConsistentRead DAX"
owner: devops-agent
objective: "Understand and correctly handle DAX consistency limitations"
context: "DAX only caches eventually consistent reads. Strongly consistent read requests are passed through to DynamoDB directly, bypassing the DAX cache entirely. This means strongly consistent reads through DAX do not benefit from caching and add network latency. Applications that require strongly consistent reads should call DynamoDB directly for those operations. Write-through DAX updates the item cache but the query cache may still serve stale results."
---

## Phase 1 — Triage

MUST:
- Check if the application uses ConsistentRead=true with DAX
- Check DAX cache hit rates to see if strongly consistent reads are bypassing cache: `aws cloudwatch get-metric-statistics --namespace AWS/DAX --metric-name ItemCacheHits --dimensions Name=ClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Verify the application's consistency requirements per operation
- Check DAX latency vs direct DynamoDB latency for strongly consistent reads

SHOULD:
- Identify which operations require strongly consistent reads
- Check if strongly consistent reads through DAX add unnecessary latency
- Review if the application can tolerate eventually consistent reads for some operations

MAY:
- Benchmark DAX eventually consistent reads vs direct DynamoDB strongly consistent reads
- Check if the application mixes DAX and direct DynamoDB calls correctly
- Review if a dual-client pattern (DAX for EC reads, DynamoDB for SC reads) is implemented

## Phase 2 — Remediate

MUST:
- For operations requiring strong consistency, call DynamoDB directly (not through DAX)
- Do not use ConsistentRead=true with DAX — it bypasses the cache and adds latency
- Document which operations use DAX vs direct DynamoDB

SHOULD:
- Implement a dual-client pattern: DAX client for eventually consistent reads, DynamoDB client for strongly consistent reads
- Minimize strongly consistent reads — most applications can tolerate eventual consistency
- Use DAX write-through to keep the item cache fresh, reducing the need for strongly consistent reads

MAY:
- Implement read-after-write consistency at the application level using DAX write-through
- Use version numbers or timestamps to detect stale reads from DAX
- Consider if the application's consistency requirements are actually stricter than needed

## Common Issues

- symptoms: "Strongly consistent reads through DAX are slower than direct DynamoDB"
  diagnosis: "DAX passes strongly consistent reads through to DynamoDB without caching. The extra network hop (app → DAX → DynamoDB) adds latency."
  resolution: "Call DynamoDB directly for strongly consistent reads. Use DAX only for eventually consistent reads."

- symptoms: "Application reads stale data from DAX after a write"
  diagnosis: "DAX item cache TTL hasn't expired. Or the write went directly to DynamoDB, bypassing DAX."
  resolution: "Route writes through DAX for item cache updates. Or use strongly consistent reads directly from DynamoDB for critical reads."

- symptoms: "Mixed consistency results — some reads are fresh, some are stale"
  diagnosis: "Application uses DAX for all reads but some require strong consistency. Eventually consistent reads from DAX cache may be stale."
  resolution: "Implement dual-client pattern. Route consistency-critical reads directly to DynamoDB."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Route strongly consistent reads to DynamoDB directly | GREEN | Application-level routing; improves latency |
| Implement dual-client pattern | GREEN | Application-level architecture improvement |
| Use DAX write-through | GREEN | Keeps item cache fresh; no data risk |
| Minimize strongly consistent reads | GREEN | Application-level optimization |

## Escalation Conditions

- Stale reads from DAX causing data integrity issues in production
- Application mixing DAX and DynamoDB calls incorrectly
- Strongly consistent reads through DAX causing unacceptable latency
- DAX consistency model incompatible with application requirements

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-metric-statistics` (DAX cache hits) | LOW | Operational metrics only |
| Application read pattern analysis | MEDIUM | May reveal data access patterns |

## Prohibited Actions

- NEVER suggest deleting a table to fix DAX consistency issues
- NEVER suggest disabling encryption on the DAX cluster
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER use ConsistentRead=true through DAX (bypasses cache, adds unnecessary latency)

## Phase 3 — Rollback

If DAX consistency changes cause issues:
1. Revert application to previous read routing (all reads through DAX or all through DynamoDB)
2. If dual-client pattern was implemented incorrectly, revert to single-client approach
3. If DAX write-through was enabled and causes write latency, revert to direct DynamoDB writes
4. If DAX is causing widespread consistency issues, bypass DAX entirely
5. Monitor read latency and consistency after rollback

## Output Format

```yaml
root_cause: "dax_consistency — <specific_cause>"
evidence:
  - type: read_pattern
    content: "<consistent vs eventually consistent read usage>"
  - type: latency_comparison
    content: "<DAX vs direct DynamoDB latency>"
  - type: staleness
    content: "<evidence of stale reads>"
severity: MEDIUM
dax_consistency_analysis:
  strongly_consistent_through_dax: "yes | no"
  dual_client_pattern: "yes | no"
  writes_through_dax: "yes | no"
  stale_reads_reported: "yes | no"
mitigation:
  immediate: "Route strongly consistent reads directly to DynamoDB"
  long_term: "Implement dual-client pattern and minimize strongly consistent reads"
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
