---
title: "B2 — NFS/SMB Share Issues"
description: "Diagnose NFS and SMB file share configuration and access problems"
status: active
severity: HIGH
triggers:
  - "NFS share error"
  - "SMB share error"
  - "mount failed"
  - "share access denied"
owner: devops-agent
objective: "Identify and resolve NFS/SMB file share configuration and access issues"
context: "File gateway shares can be NFS (v3, v4.1) or SMB (v2, v3). NFS shares use IP-based client access lists. SMB shares use Active Directory or guest access. Each share maps to an S3 bucket with optional prefix. Share-level settings control squash behavior, read-only mode, and object storage class."
---

## Phase 1 — Triage

MUST:
- Check NFS share config: `aws storagegateway describe-nfs-file-shares --file-share-arn-list <arn>`
- Check SMB share config: `aws storagegateway describe-smb-file-shares --file-share-arn-list <arn>`
- Verify client can reach gateway on NFS port (2049) or SMB port (445)
- Check file share status is AVAILABLE

SHOULD:
- For NFS: verify client IP is in allowed client list
- For SMB: verify AD join status or guest access configuration
- Check S3 bucket IAM role permissions

MAY:
- Test mount from client: `mount -t nfs <gateway-ip>:/<share-name> /mnt/test`
- Check SMB settings: `aws storagegateway describe-smb-settings --gateway-arn <arn>`

## Phase 2 — Remediate

MUST:
- For NFS: add client IP to allowed list: `aws storagegateway update-nfs-file-share --file-share-arn <arn> --client-list "10.0.0.0/8"`
- For SMB: verify AD join or enable guest access
- Ensure gateway security group allows inbound on port 2049 (NFS) or 445 (SMB)

SHOULD:
- Configure appropriate squash settings for NFS (RootSquash, NoSquash, AllSquash)
- Set correct default storage class for new objects

MAY:
- Enable audit logging for file share access
- Configure notification policy for file upload events

## Common Issues

- symptoms: "NFS mount: access denied by server"
  diagnosis: "Client IP not in file share's allowed client list."
  resolution: "Add client IP/CIDR to NFS file share client list."

- symptoms: "SMB share: authentication failed"
  diagnosis: "Gateway not joined to AD or guest access not enabled."
  resolution: "Join gateway to AD domain or enable guest access with password."

- symptoms: "Files created with wrong permissions"
  diagnosis: "NFS squash settings overriding client uid/gid."
  resolution: "Adjust squash settings: NoSquash preserves client identity."

## Output Format

```yaml
root_cause: "nfs_smb_share — <specific_cause>"
evidence:
  - type: share_config
    content: "<NFS/SMB share configuration>"
  - type: client_access
    content: "<client list or AD status>"
  - type: connectivity
    content: "<port connectivity test>"
severity: HIGH
mitigation:
  immediate: "Fix share access configuration"
  long_term: "Document share access requirements and automate configuration"
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
