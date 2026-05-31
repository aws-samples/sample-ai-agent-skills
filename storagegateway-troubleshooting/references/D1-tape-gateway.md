---
title: "D1 — Tape Gateway Issues"
description: "Diagnose tape gateway operational problems"
status: active
severity: HIGH
triggers:
  - "tape gateway error"
  - "VTL not accessible"
  - "tape backup failed"
  - "media changer error"
owner: devops-agent
objective: "Identify and resolve tape gateway operational issues"
context: "Tape gateways present a virtual tape library (VTL) to backup applications via iSCSI. Virtual tapes are stored in S3. Archived tapes go to S3 Glacier or Deep Archive. The gateway emulates a tape library with media changer and tape drives. Backup software must be compatible with the emulated hardware."
---

## Phase 1 — Triage

MUST:
- Check gateway status: `aws storagegateway describe-gateway-information --gateway-arn <gateway-arn>`
- List tapes: `aws storagegateway list-tapes --gateway-arn <gateway-arn>`
- Check tape status: `aws storagegateway describe-tapes --gateway-arn <gateway-arn>`
- Verify iSCSI connectivity to VTL on port 3260

SHOULD:
- Check cache and upload buffer: `aws storagegateway describe-cache --gateway-arn <gateway-arn>`
- Verify backup software recognizes the VTL and tape drives
- Check for tapes in error state

MAY:
- Review backup software logs for tape operation errors
- Check CloudWatch metrics: CacheHitPercent, CloudBytesUploaded

## Phase 2 — Remediate

MUST:
- Ensure iSCSI targets are discovered by backup software
- Verify cache and upload buffer have sufficient capacity
- Check tape status — tapes in error state may need to be deleted and recreated

SHOULD:
- Create new tapes if needed: `aws storagegateway create-tapes --gateway-arn <arn> --tape-size-in-bytes <size> --client-token <token> --num-tapes-to-create 1 --tape-barcode-prefix <prefix>`
- Configure backup software to use correct media changer and tape drive types

MAY:
- Optimize tape size based on backup job sizes
- Configure tape pool for archival (Glacier vs Deep Archive)

## Common Issues

- symptoms: "Backup software cannot find tape library"
  diagnosis: "iSCSI targets not discovered or wrong device type configured."
  resolution: "Discover iSCSI targets on port 3260. Configure correct media changer type in backup software."

- symptoms: "Tape backup fails midway"
  diagnosis: "Upload buffer full or cache exhaustion."
  resolution: "Check UploadBufferPercentUsed. Add upload buffer capacity. Check bandwidth."

- symptoms: "No available tapes in library"
  diagnosis: "All tapes used, archived, or in error state."
  resolution: "Create new virtual tapes. Check for tapes in error state."

## Output Format

```yaml
root_cause: "tape_gateway — <specific_cause>"
evidence:
  - type: gateway_status
    content: "<gateway operational state>"
  - type: tape_inventory
    content: "<tape count, status, capacity>"
  - type: iscsi_connectivity
    content: "<VTL iSCSI target status>"
severity: HIGH
mitigation:
  immediate: "Fix the specific tape gateway issue"
  long_term: "Monitor tape inventory and cache/buffer capacity"
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
