---
title: "C1 — Volume Gateway Issues"
description: "Diagnose volume gateway operational problems"
status: active
severity: HIGH
triggers:
  - "volume gateway error"
  - "iSCSI connection failed"
  - "volume not available"
  - "volume read/write error"
owner: devops-agent
objective: "Identify and resolve volume gateway operational issues"
context: "Volume gateways provide iSCSI block storage backed by S3. Volumes can be cached (primary data in S3) or stored (primary data local). iSCSI initiators on application servers connect to volumes. Common issues include iSCSI connectivity, volume health, cache/upload buffer exhaustion, and snapshot operations."
---

## Phase 1 — Triage

MUST:
- List volumes: `aws storagegateway list-volumes --gateway-arn <gateway-arn>`
- Check volume details: `aws storagegateway describe-cached-iscsi-volumes --volume-arns <volume-arns>` or `aws storagegateway describe-stored-iscsi-volumes --volume-arns <volume-arns>`
- Check cache and upload buffer: `aws storagegateway describe-cache --gateway-arn <gateway-arn>`
- Verify iSCSI target connectivity from application server

SHOULD:
- Check CloudWatch metrics: CacheHitPercent, ReadBytes, WriteBytes, VolumeSafeToDetach
- Verify CHAP authentication if configured
- Check volume recovery points

MAY:
- Review iSCSI initiator logs on the application server
- Check for volume I/O errors in gateway logs

## Phase 2 — Remediate

MUST:
- Ensure iSCSI target is accessible on port 3260 from application server
- Verify cache and upload buffer have sufficient space
- Check volume status is AVAILABLE

SHOULD:
- Reconnect iSCSI initiator if connection was lost
- Add cache/upload buffer capacity if metrics show exhaustion

MAY:
- Create volume snapshot before troubleshooting: `aws storagegateway create-snapshot --volume-arn <arn> --snapshot-description "pre-troubleshooting"`
- Optimize iSCSI settings (queue depth, timeout values)

## Common Issues

- symptoms: "iSCSI target not found"
  diagnosis: "Gateway offline or port 3260 blocked."
  resolution: "Check gateway status. Verify port 3260 is open between client and gateway."

- symptoms: "Volume I/O very slow"
  diagnosis: "Cache miss rate high or upload buffer full."
  resolution: "Check CacheHitPercent. Add cache capacity. Check upload bandwidth."

- symptoms: "Volume in IRRECOVERABLE state"
  diagnosis: "Volume data corruption or gateway failure."
  resolution: "Restore from latest snapshot. Create new volume from snapshot."

## Output Format

```yaml
root_cause: "volume_gateway — <specific_cause>"
evidence:
  - type: volume_status
    content: "<volume state and health>"
  - type: iscsi_connectivity
    content: "<iSCSI connection status>"
  - type: cache_metrics
    content: "<cache hit rate and buffer usage>"
severity: HIGH
mitigation:
  immediate: "Fix the specific volume gateway issue"
  long_term: "Monitor volume health and cache metrics"
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
