---
title: "D2 — Virtual Tape Issues"
description: "Diagnose virtual tape creation, archival, and retrieval problems"
status: active
severity: MEDIUM
triggers:
  - "virtual tape error"
  - "tape archival failed"
  - "tape retrieval"
  - "tape stuck"
owner: devops-agent
objective: "Identify and resolve virtual tape lifecycle issues"
context: "Virtual tapes go through lifecycle states: CREATING → AVAILABLE → IN USE → ARCHIVING → ARCHIVED. Tapes must be ejected before archiving. Archived tapes go to S3 Glacier (3-5 hour retrieval) or Deep Archive (12 hour retrieval). Tape size range is 100 GB to 5 TB. Maximum 1,500 tapes per gateway."
---

## Phase 1 — Triage

MUST:
- Check tape status: `aws storagegateway describe-tapes --gateway-arn <gateway-arn> --tape-arns <tape-arn>`
- List tapes by status: `aws storagegateway list-tapes --gateway-arn <gateway-arn>`
- Check tape count against limit (1,500 per gateway)
- For archived tapes: `aws storagegateway describe-tape-archives --tape-arns <tape-arn>`

SHOULD:
- Check if tape is stuck in ARCHIVING state (upload in progress)
- Verify tape pool configuration (Glacier vs Deep Archive)
- Check upload buffer utilization during archival

MAY:
- Review backup software tape management logs
- Check CloudWatch metrics for tape operations

## Phase 2 — Remediate

MUST:
- For stuck tapes: check upload buffer and bandwidth
- For retrieval: `aws storagegateway retrieve-tape-archive --tape-arn <tape-arn> --gateway-arn <gateway-arn>`
- Allow retrieval time: 3-5 hours (Glacier), 12 hours (Deep Archive)

SHOULD:
- Delete tapes in error state and create replacements
- Monitor archival progress via CloudWatch metrics

MAY:
- Optimize tape pool selection based on retrieval time requirements
- Implement tape rotation policies in backup software

## Common Issues

- symptoms: "Tape stuck in ARCHIVING state"
  diagnosis: "Upload buffer full or insufficient bandwidth for tape data upload."
  resolution: "Check upload buffer. Increase bandwidth. Wait for upload to complete."

- symptoms: "Cannot access archived tape"
  diagnosis: "Archived tapes must be retrieved before access."
  resolution: "Initiate retrieval. Wait 3-5 hours (Glacier) or 12 hours (Deep Archive)."

- symptoms: "Tape creation fails"
  diagnosis: "Tape count at limit (1,500) or invalid tape size."
  resolution: "Delete unused tapes. Tape size must be 100 GB to 5 TB."

## Output Format

```yaml
root_cause: "virtual_tape — <specific_cause>"
evidence:
  - type: tape_status
    content: "<tape lifecycle state>"
  - type: tape_pool
    content: "<Glacier or Deep Archive>"
  - type: upload_progress
    content: "<archival upload status>"
severity: MEDIUM
mitigation:
  immediate: "Fix the specific tape lifecycle issue"
  long_term: "Implement tape management policies and monitoring"
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
