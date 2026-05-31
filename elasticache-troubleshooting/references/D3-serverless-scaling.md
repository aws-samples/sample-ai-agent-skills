---
title: "D3 — ElastiCache Serverless Scaling Issues"
description: "Diagnose scaling, limits, and performance issues with ElastiCache Serverless"
status: active
severity: MEDIUM
triggers:
  - "ElastiCache Serverless"
  - "serverless scaling"
  - "ECPU"
  - "serverless limits"
  - "serverless throttling"
owner: devops-agent
objective: "Resolve ElastiCache Serverless scaling issues and optimize for the serverless pricing model"
context: "ElastiCache Serverless automatically scales compute and memory without node management. Pricing is based on data stored (GB-hours) and ElastiCache Processing Units (ECPUs). Serverless has different limits than provisioned clusters — maximum data storage, connection limits, and some Redis/Memcached commands may not be supported. Scaling is automatic but has upper bounds that can be configured."
---

## Phase 1 — Triage

MUST:
- Check serverless cache status: `aws elasticache describe-serverless-caches --serverless-cache-name <cache-name>`
- Check ECPU usage: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name ElastiCacheProcessingUnits --dimensions Name=ServerlessCacheName,Value=<cache-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check data storage: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name BytesUsedForCache --dimensions Name=ServerlessCacheName,Value=<cache-name> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check throttled requests: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name ThrottledCmds --dimensions Name=ServerlessCacheName,Value=<cache-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check current scaling limits: `aws elasticache describe-serverless-caches --serverless-cache-name <cache-name> --query 'ServerlessCaches[*].CacheUsageLimits'`

SHOULD:
- Check if maximum ECPU or storage limits are configured too low
- Verify the commands being used are supported in Serverless mode
- Check connection count: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name CurrConnections --dimensions Name=ServerlessCacheName,Value=<cache-name> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Review recent events: `aws elasticache describe-events --source-type serverless-cache --duration 1440`

MAY:
- Compare ECPU consumption patterns to identify optimization opportunities
- Check if specific commands consume disproportionate ECPUs
- Review AWS documentation for current Serverless limits

## Phase 2 — Remediate

MUST:
- Increase maximum ECPU or storage limits if throttling occurs: `aws elasticache modify-serverless-cache --serverless-cache-name <cache-name> --cache-usage-limits 'DataStorage={Maximum=<gb>,Unit=GB},ECPUPerSecond={Maximum=<ecpu>}'`
- Verify all commands used are supported in Serverless mode
- Optimize high-ECPU commands to reduce consumption

SHOULD:
- Set appropriate maximum limits to control costs while avoiding throttling
- Use connection pooling to stay within connection limits
- Monitor ThrottledCmds metric with CloudWatch alarms

MAY:
- Consider provisioned clusters if Serverless costs exceed provisioned for steady workloads
- Implement client-side caching to reduce ECPU consumption
- Use batch operations to reduce per-command overhead

## Common Issues

- symptoms: "ThrottledCmds increasing, requests being rejected"
  diagnosis: "ECPU per second limit is too low for the workload."
  resolution: "Increase the maximum ECPUPerSecond limit using modify-serverless-cache."

- symptoms: "Unsupported command error"
  diagnosis: "Some Redis commands are not supported in ElastiCache Serverless."
  resolution: "Check AWS documentation for supported commands. Use alternative commands or switch to provisioned."

- symptoms: "Unexpected high costs with Serverless"
  diagnosis: "ECPU consumption is higher than expected due to complex commands or high throughput."
  resolution: "Optimize commands, implement caching, or evaluate provisioned clusters for cost comparison."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Increase maximum ECPU limit | GREEN | Configuration change; prevents throttling |
| Increase maximum storage limit | GREEN | Configuration change; allows data growth |
| Use connection pooling | GREEN | Application-level optimization |
| Optimize high-ECPU commands | GREEN | Application-level improvement |

## Escalation Conditions

- ThrottledCmds increasing on a production serverless cache
- Unsupported command errors blocking application functionality
- Unexpected high costs requiring immediate cost control
- Serverless cache limits insufficient for workload requirements
- Connection limits reached on serverless cache

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-serverless-caches` | MEDIUM | Exposes cache configuration and limits |
| `get-metric-statistics` (ECPU, storage) | LOW | Operational metrics only |
| `describe-events` | LOW | Operational events only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to reduce ECPU usage
- NEVER suggest disabling AUTH on serverless caches
- NEVER suggest disabling encryption in transit on serverless caches
- NEVER suggest reducing node count during peak traffic (not applicable to serverless, but do not suggest migrating to smaller provisioned during peak)
- NEVER set maximum ECPU or storage limits to 0

## Phase 3 — Rollback

If serverless scaling changes cause issues:
1. Revert ECPU and storage limits to previous values: `aws elasticache modify-serverless-cache --serverless-cache-name <cache-name> --cache-usage-limits 'DataStorage={Maximum=<previous-gb>,Unit=GB},ECPUPerSecond={Maximum=<previous-ecpu>}'`
2. If connection pooling was implemented and causes issues, revert to previous connection management
3. If command optimizations cause functional issues, revert application code
4. If costs are too high, reduce limits gradually while monitoring ThrottledCmds
5. Consider migrating to provisioned clusters if serverless costs are consistently higher

## Output Format

```yaml
root_cause: "serverless_scaling — <specific_cause>"
evidence:
  - type: serverless_config
    content: "<cache usage limits and status>"
  - type: ecpu_usage
    content: "<ECPU consumption metrics>"
  - type: throttled_cmds
    content: "<throttled command count>"
severity: MEDIUM
mitigation:
  immediate: "Increase ECPU or storage limits to stop throttling"
  long_term: "Optimize command patterns and set appropriate scaling limits"
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
