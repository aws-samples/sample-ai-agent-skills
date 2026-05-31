---
title: "E1 — Cache Performance Issues"
description: "Diagnose cache hit rate degradation and cache-related performance problems"
status: active
severity: MEDIUM
triggers:
  - "cache performance"
  - "cache hit rate low"
  - "slow reads"
  - "cache miss"
owner: devops-agent
objective: "Identify and resolve cache performance issues on Storage Gateway"
context: "Cache performance directly impacts read latency. Low cache hit rates mean more reads go to S3 (high latency). Cache size should accommodate the active working set. Cache disks should be SSD. CachePercentDirty indicates pending uploads. CachePercentUsed near 100% means evictions are occurring."
---

## Phase 1 — Triage

MUST:
- Check cache metrics: `aws cloudwatch get-metric-statistics --namespace AWS/StorageGateway --metric-name CacheHitPercent --dimensions Name=GatewayId,Value=<gw-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check cache utilization: `aws storagegateway describe-cache --gateway-arn <gateway-arn>`
- Check CachePercentDirty and CachePercentUsed metrics
- Verify cache disk type (SSD vs HDD)

SHOULD:
- Analyze read/write patterns to understand working set size
- Check if cache disk I/O is saturated
- Compare cache size to active data volume

MAY:
- Check gateway VM disk I/O metrics
- Review access patterns for optimization opportunities

## Phase 2 — Remediate

MUST:
- Add cache disk capacity if CachePercentUsed is consistently > 80%
- Use SSD-backed disks for cache (not HDD)
- Add cache: `aws storagegateway add-cache --gateway-arn <arn> --disk-ids <disk-id>`

SHOULD:
- Size cache to at least 20% of total data for typical workloads
- Monitor CachePercentDirty — high values indicate upload bottleneck
- Increase upload bandwidth if dirty cache is growing

MAY:
- Optimize application access patterns to improve locality
- Consider read-only file shares for read-heavy workloads

## Common Issues

- symptoms: "CacheHitPercent below 50%"
  diagnosis: "Cache too small for working set or random access pattern."
  resolution: "Add cache capacity. Size cache for active working set."

- symptoms: "CachePercentDirty consistently high"
  diagnosis: "Upload bandwidth insufficient to flush dirty cache."
  resolution: "Increase upload bandwidth. Check bandwidth rate limits."

- symptoms: "Read latency spikes"
  diagnosis: "Cache evictions causing reads from S3."
  resolution: "Add cache capacity. Check CachePercentUsed."

## Output Format

```yaml
root_cause: "cache_performance — <specific_cause>"
evidence:
  - type: cache_hit_rate
    content: "<CacheHitPercent over time>"
  - type: cache_utilization
    content: "<CachePercentUsed and CachePercentDirty>"
  - type: cache_config
    content: "<cache disk size and type>"
severity: MEDIUM
mitigation:
  immediate: "Add cache capacity or optimize access patterns"
  long_term: "Right-size cache for workload and monitor continuously"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Gateway volumes contain production application data
- Tape archives may contain regulated backup data
- File shares expose S3 bucket contents to on-premises systems
- Classification: CONFIDENTIAL - production data in transit and at rest

## Prohibited Actions

- NEVER shut down or reboot gateway VMs during active data transfers
- NEVER delete volumes or tapes containing unverified backup data
- NEVER modify cache disk configuration on production gateways without maintenance window
- NEVER change iSCSI targets or NFS/SMB shares during business hours without approval

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
