---
title: "B4 — ElastiCache Slow Commands"
description: "Diagnose and optimize slow Redis commands identified in SLOWLOG or causing performance degradation"
status: active
severity: HIGH
triggers:
  - "slow command"
  - "SLOWLOG"
  - "KEYS command"
  - "O(N) command"
  - "blocking command"
  - "slow query"
owner: devops-agent
objective: "Identify slow commands and replace them with efficient alternatives"
context: "Redis is single-threaded for command execution. A single slow command blocks all other commands. SLOWLOG records commands exceeding slowlog-log-slower-than (default 10,000 microseconds = 10ms). Common offenders: KEYS (O(N) scan of all keys), SORT, SMEMBERS/HGETALL on large collections, LRANGE on large lists, and Lua scripts with heavy computation."
---

## Phase 1 — Triage

MUST:
- Get slow log entries: `redis-cli -h <endpoint> -p 6379 SLOWLOG GET 50`
- Check slow log configuration: `redis-cli -h <endpoint> -p 6379 CONFIG GET slowlog-log-slower-than`
- Check slow log length: `redis-cli -h <endpoint> -p 6379 SLOWLOG LEN`
- Check command statistics: `redis-cli -h <endpoint> -p 6379 INFO commandstats`
- Correlate with EngineCPU: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name EngineCPUUtilization --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`

SHOULD:
- Identify large keys that slow commands operate on: `redis-cli -h <endpoint> -p 6379 --bigkeys`
- Check for KEYS command usage in commandstats (should be zero in production)
- Check Lua script execution time: `redis-cli -h <endpoint> -p 6379 INFO stats | grep lua`
- Review client list for blocked clients: `redis-cli -h <endpoint> -p 6379 CLIENT LIST`

MAY:
- Enable latency monitoring: `redis-cli -h <endpoint> -p 6379 CONFIG SET latency-monitor-threshold 50`
- Check MEMORY USAGE for specific large keys: `redis-cli -h <endpoint> -p 6379 MEMORY USAGE <key>`
- Lower slowlog-log-slower-than to capture more commands: `redis-cli -h <endpoint> -p 6379 CONFIG SET slowlog-log-slower-than 5000`

## Phase 2 — Remediate

MUST:
- Replace KEYS with SCAN (cursor-based iteration, non-blocking)
- Replace SMEMBERS with SSCAN for large sets
- Replace HGETALL with HSCAN for large hashes
- Replace large LRANGE with paginated LRANGE (small ranges)
- Optimize or break up long-running Lua scripts

SHOULD:
- Rename KEYS command to prevent accidental use: use rename-commands in parameter group
- Set appropriate slowlog-log-slower-than (5000-10000 microseconds recommended)
- Break large data structures into smaller ones (e.g., hash with 100K fields → multiple hashes)

MAY:
- Use OBJECT ENCODING to check data structure encoding efficiency
- Consider using Redis Streams instead of large lists for queue patterns
- Implement client-side caching to reduce command volume

## Common Issues

- symptoms: "SLOWLOG shows KEYS * taking 500ms+"
  diagnosis: "KEYS scans all keys in the database — O(N) complexity."
  resolution: "Replace with SCAN 0 MATCH <pattern> COUNT 100. Never use KEYS in production."

- symptoms: "HGETALL on hash with 50,000+ fields in SLOWLOG"
  diagnosis: "Large hash retrieval is O(N) where N is the number of fields."
  resolution: "Use HSCAN for iteration. Retrieve only needed fields with HMGET. Split into smaller hashes."

- symptoms: "Lua script execution exceeding 5 seconds"
  diagnosis: "Complex Lua script blocking the event loop."
  resolution: "Optimize the script. Break into smaller operations. Consider using Redis Functions (7.x)."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Replace KEYS with SCAN | GREEN | Application-level improvement; non-blocking alternative |
| Replace SMEMBERS with SSCAN | GREEN | Application-level improvement; paginated iteration |
| Replace HGETALL with HSCAN | GREEN | Application-level improvement; paginated iteration |
| Rename KEYS command | YELLOW | Parameter group change; requires reboot; blocks KEYS permanently |
| Break large data structures | YELLOW | Application-level redesign; requires code changes |
| Lower slowlog-log-slower-than | GREEN | Diagnostic configuration; captures more commands |

## Escalation Conditions

- Slow command blocking the event loop for >5 seconds in production
- KEYS command being used in production application code
- Lua script execution exceeding 5 seconds
- Slow commands causing cascading timeouts across dependent services
- Large data structures (>100K elements) being accessed with O(N) commands

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `SLOWLOG GET` | MEDIUM | Exposes command patterns, key names, and arguments |
| `INFO commandstats` | LOW | Command frequency statistics only |
| `--bigkeys` | MEDIUM | Exposes key names and sizes |
| `CLIENT LIST` | MEDIUM | Exposes client connection details |
| `MEMORY USAGE <key>` | MEDIUM | Exposes specific key memory consumption |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix slow command issues
- NEVER suggest disabling AUTH to reduce command overhead
- NEVER suggest disabling encryption in transit to reduce TLS processing
- NEVER suggest reducing node count during peak traffic
- NEVER use KEYS, SMEMBERS, or HGETALL on large collections in production as diagnostic commands

## Phase 3 — Rollback

If slow command remediation causes issues:
1. Revert application code to previous command patterns if SCAN-family replacements cause functional issues
2. If KEYS command was renamed in parameter group, revert the rename and reboot
3. If data structures were split, revert application code to use original large structures
4. If slowlog-log-slower-than was changed, revert to previous threshold
5. Monitor SLOWLOG and EngineCPU to verify improvements after rollback

## Output Format

```yaml
root_cause: "slow_commands — <specific_cause>"
evidence:
  - type: slow_log
    content: "<SLOWLOG entries with command, duration, timestamp>"
  - type: command_stats
    content: "<command frequency and avg microseconds>"
  - type: big_keys
    content: "<large keys identified>"
severity: HIGH
mitigation:
  immediate: "Replace slow O(N) commands with SCAN-family alternatives"
  long_term: "Implement command auditing, rename dangerous commands, optimize data structures"
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
