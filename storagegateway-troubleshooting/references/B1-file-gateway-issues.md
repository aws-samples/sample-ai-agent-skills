---
title: "B1 — File Gateway Issues"
description: "Diagnose general file gateway operational problems"
status: active
severity: HIGH
triggers:
  - "file gateway error"
  - "file share not accessible"
  - "cannot read files"
  - "write failures on file share"
owner: devops-agent
objective: "Identify and resolve file gateway operational issues"
context: "File gateways provide NFS/SMB access to S3 storage. Files are stored as S3 objects with metadata preserved. Common issues include cache exhaustion, S3 bucket access problems, file share configuration errors, and client connectivity issues."
---

## Phase 1 — Triage

MUST:
- Check gateway status: `aws storagegateway describe-gateway-information --gateway-arn <gateway-arn>`
- List file shares: `aws storagegateway list-file-shares --gateway-arn <gateway-arn>`
- Check cache health: `aws storagegateway describe-cache --gateway-arn <gateway-arn>`
- Check CloudWatch metrics: CacheHitPercent, CachePercentDirty, CachePercentUsed

SHOULD:
- Verify S3 bucket is accessible from gateway IAM role
- Check file share status and configuration
- Review CloudWatch Logs for file gateway errors

MAY:
- Check client-side mount configuration
- Verify network path between client and gateway

## Phase 2 — Remediate

MUST:
- Ensure cache disks are healthy and have sufficient space
- Verify S3 bucket permissions for the gateway's IAM role
- Check file share is in AVAILABLE state

SHOULD:
- Add cache disk capacity if CachePercentUsed is consistently high
- Monitor CachePercentDirty — high values indicate upload bottleneck

MAY:
- Optimize file access patterns to improve cache hit rate
- Configure read-only shares for read-heavy workloads

## Common Issues

- symptoms: "File writes failing or very slow"
  diagnosis: "Cache or upload buffer full."
  resolution: "Check CachePercentDirty and UploadBufferPercentUsed. Add disk capacity or increase bandwidth."

- symptoms: "Files not visible on share"
  diagnosis: "Cache not refreshed after direct S3 changes."
  resolution: "Run cache refresh: aws storagegateway refresh-cache --file-share-arn <arn>"

- symptoms: "Permission denied accessing file share"
  diagnosis: "NFS/SMB client permissions or file share access configuration."
  resolution: "Check file share client list (NFS) or AD permissions (SMB)."

## Output Format

```yaml
root_cause: "file_gateway_issue — <specific_cause>"
evidence:
  - type: gateway_status
    content: "<gateway operational state>"
  - type: cache_health
    content: "<cache metrics>"
  - type: file_share_status
    content: "<file share configuration and state>"
severity: HIGH
mitigation:
  immediate: "Fix the specific file gateway issue"
  long_term: "Monitor cache health and optimize file access patterns"
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
