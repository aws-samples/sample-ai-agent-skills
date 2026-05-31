---
title: "C2 — Cached vs Stored Volume Issues"
description: "Diagnose issues specific to cached or stored volume modes"
status: active
severity: MEDIUM
triggers:
  - "cached volume"
  - "stored volume"
  - "volume mode"
  - "local storage full"
owner: devops-agent
objective: "Identify and resolve issues specific to cached or stored volume modes"
context: "Cached volumes store primary data in S3 with frequently accessed data cached locally. Stored volumes store all data locally with async backup to S3. Mode cannot be changed after creation. Each mode has different failure modes, capacity planning, and performance characteristics."
---

## Phase 1 — Triage

MUST:
- Identify volume mode: `aws storagegateway describe-cached-iscsi-volumes --volume-arns <arn>` (cached) or `aws storagegateway describe-stored-iscsi-volumes --volume-arns <arn>` (stored)
- For cached: check CacheHitPercent, CachePercentDirty, CachePercentUsed
- For stored: check local disk usage and upload progress
- Check volume status and attachment state

SHOULD:
- For cached: verify cache disk size is appropriate for working set
- For stored: verify local disk has capacity for full volume
- Check upload buffer utilization for both modes

MAY:
- Analyze I/O patterns to determine if correct mode is being used
- Compare latency metrics between cached and stored volumes

## Phase 2 — Remediate

MUST:
- For cached volumes with low cache hit: add cache disk capacity
- For stored volumes with disk full: expand local storage or reduce volume size
- Monitor upload buffer to prevent write stalls

SHOULD:
- For cached: ensure cache is large enough for active working set (20% of volume recommended)
- For stored: ensure local disk is at least as large as the volume

MAY:
- Consider migrating to the other mode if current mode doesn't fit workload
- Use EBS snapshots to create volumes in the correct mode

## Common Issues

- symptoms: "Cached volume: high latency on reads"
  diagnosis: "Cache miss rate high — working set exceeds cache size."
  resolution: "Add cache disk capacity. Cache should be 20%+ of volume size for active data."

- symptoms: "Stored volume: local disk full"
  diagnosis: "Volume data exceeds local disk capacity."
  resolution: "Expand local storage. Stored volumes require local disk >= volume size."

- symptoms: "Want to change volume from cached to stored"
  diagnosis: "Volume mode cannot be changed after creation."
  resolution: "Create new stored volume. Restore data from snapshot. Delete old cached volume."

## Output Format

```yaml
root_cause: "volume_mode_issue — <specific_cause>"
evidence:
  - type: volume_mode
    content: "<cached or stored>"
  - type: capacity_metrics
    content: "<cache/disk utilization>"
  - type: performance_metrics
    content: "<latency and throughput>"
severity: MEDIUM
mitigation:
  immediate: "Address capacity or performance issue for current mode"
  long_term: "Right-size cache/storage and evaluate if correct mode is used"
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
