---
title: "H3 — ElastiCache Cross-Slot Command Errors"
description: "Diagnose CROSSSLOT errors in Redis cluster mode enabled"
status: active
severity: MEDIUM
triggers:
  - "CROSSSLOT"
  - "cross-slot"
  - "keys in different slots"
  - "multi-key error"
  - "MGET error"
  - "pipeline error"
owner: devops-agent
objective: "Resolve CROSSSLOT errors by ensuring multi-key operations target the same hash slot"
context: "In Redis cluster mode enabled, each key is assigned to one of 16,384 hash slots. Multi-key commands (MGET, MSET, SUNION, pipeline with multiple keys, Lua scripts with multiple keys, transactions) require all keys to be in the SAME hash slot. If keys are in different slots, Redis returns a CROSSSLOT error. Hash tags {tag} force keys to the same slot by hashing only the content within braces."
---

## Phase 1 — Triage

MUST:
- Identify the failing command and keys from application logs
- Check which slots the keys belong to: `redis-cli -h <endpoint> -p 6379 CLUSTER KEYSLOT <key1>` and `redis-cli -h <endpoint> -p 6379 CLUSTER KEYSLOT <key2>`
- Verify cluster mode is enabled: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].ClusterEnabled'`
- Check the command that caused the error (MGET, MSET, SUNION, transaction, Lua script)
- Verify the application is using a Redis Cluster-aware client

SHOULD:
- Review the key naming convention for hash tag usage
- Check if the application uses transactions (MULTI/EXEC) across slots
- Check if Lua scripts reference keys in multiple slots
- Review pipeline operations for cross-slot key access

MAY:
- Analyze key distribution across slots: `redis-cli -h <endpoint> -p 6379 CLUSTER COUNTKEYSINSLOT <slot>`
- Check if the application can be refactored to use hash tags
- Review if cluster mode disabled would be more appropriate for the workload

## Phase 2 — Remediate

MUST:
- Use hash tags to ensure related keys are in the same slot: `user:{123}:profile`, `user:{123}:sessions` (both hash to slot of "123")
- Refactor multi-key operations to operate on keys in the same slot
- For Lua scripts, ensure all KEYS[] arguments are in the same slot

SHOULD:
- Design key naming conventions with hash tags from the start
- Replace cross-slot MGET with individual GET commands (or pipeline per slot)
- Use client-side aggregation instead of server-side multi-key operations

MAY:
- Consider cluster mode disabled if the workload heavily relies on multi-key operations
- Implement application-level sharding with hash tag awareness
- Use Redis Streams or pub/sub as alternatives to cross-slot operations

## Common Issues

- symptoms: "CROSSSLOT Keys in request don't hash to the same slot"
  diagnosis: "MGET or MSET with keys that hash to different slots."
  resolution: "Use hash tags: {user:123}:name, {user:123}:email. Or split into per-slot operations."

- symptoms: "Lua script fails with CROSSSLOT error"
  diagnosis: "Lua script KEYS[] arguments are in different hash slots."
  resolution: "Ensure all KEYS[] in the Lua script use the same hash tag. Pass keys in the same slot."

- symptoms: "Transaction (MULTI/EXEC) fails with CROSSSLOT"
  diagnosis: "Commands in the transaction operate on keys in different slots."
  resolution: "Use hash tags for all keys in the transaction. Or split into separate transactions per slot."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Use hash tags for related keys | GREEN | Application-level key naming convention change |
| Refactor multi-key operations | GREEN | Application-level code change |
| Split cross-slot MGET into per-slot operations | GREEN | Application-level optimization |
| Consider cluster mode disabled | RED | Requires new cluster creation and data migration |

## Escalation Conditions

- CROSSSLOT errors affecting critical production operations
- Application heavily relies on multi-key operations incompatible with cluster mode
- Hash tag implementation requires significant application refactoring
- Lua scripts with cross-slot key access in production
- Transaction (MULTI/EXEC) failures due to cross-slot keys

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `CLUSTER KEYSLOT <key>` | MEDIUM | Exposes key names |
| `describe-replication-groups` (ClusterEnabled) | LOW | Cluster mode status only |
| Application logs (failing commands) | MEDIUM | May contain key names and command arguments |
| `CLUSTER COUNTKEYSINSLOT` | LOW | Key count per slot only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix cross-slot errors
- NEVER suggest disabling AUTH to troubleshoot cross-slot issues
- NEVER suggest disabling encryption in transit to simplify cluster mode
- NEVER suggest reducing node count during peak traffic
- NEVER manually reassign slots using CLUSTER commands on managed ElastiCache

## Phase 3 — Rollback

If cross-slot remediation causes issues:
1. Revert application code to previous key naming convention if hash tags cause issues
2. If multi-key operations were split into individual operations, revert if performance is unacceptable
3. If a new cluster mode disabled cluster was created, redirect traffic back to the original cluster
4. If Lua scripts were modified, revert to previous script versions
5. Verify application functionality after rollback

## Output Format

```yaml
root_cause: "cross_slot — <specific_cause>"
evidence:
  - type: key_slots
    content: "<key-to-slot mapping>"
  - type: failing_command
    content: "<command and keys causing CROSSSLOT>"
  - type: cluster_mode
    content: "<cluster mode enabled confirmation>"
severity: MEDIUM
mitigation:
  immediate: "Use hash tags or split multi-key operations by slot"
  long_term: "Design key naming conventions with hash tags for cluster mode"
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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest disabling AUTH"
  - "NEVER suggest public subnet placement"
