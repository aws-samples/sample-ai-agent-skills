---
title: "G2 — Recovery Issues"
description: "Diagnose data recovery and gateway recovery problems"
status: active
severity: HIGH
triggers:
  - "recovery failed"
  - "restore from snapshot"
  - "gateway recovery"
  - "data loss"
owner: devops-agent
objective: "Identify and resolve data and gateway recovery issues"
context: "Recovery scenarios include restoring volumes from EBS snapshots, recovering a failed gateway, and retrieving archived tapes. Volume recovery creates a new volume from a snapshot. Gateway recovery involves deploying a new gateway VM and associating existing resources. Tape retrieval from Glacier has time delays."
---

## Phase 1 — Triage

MUST:
- Identify recovery scenario: volume restore, gateway recovery, or tape retrieval
- For volume restore: check available snapshots: `aws ec2 describe-snapshots --filters Name=volume-id,Values=<volume-id>`
- For gateway recovery: check if gateway resources (shares/volumes/tapes) still exist in AWS
- For tape retrieval: check tape archive status: `aws storagegateway describe-tape-archives`

SHOULD:
- Verify recovery target (new gateway or existing gateway)
- Check IAM permissions for recovery operations
- Estimate recovery time based on data volume

MAY:
- Review disaster recovery procedures
- Check if cross-region snapshots exist for DR

## Phase 2 — Remediate

MUST:
- For volume restore: create volume from snapshot: `aws storagegateway create-cached-iscsi-volume --gateway-arn <arn> --volume-size-in-bytes <size> --snapshot-id <snap-id> --target-name <target> --network-interface-id <eni-id>`
- For gateway recovery: deploy new gateway VM, activate, and reassociate resources
- For tape retrieval: `aws storagegateway retrieve-tape-archive --tape-arn <arn> --gateway-arn <arn>`

SHOULD:
- Test recovery procedures regularly
- Document recovery time objectives (RTO) and recovery point objectives (RPO)

MAY:
- Implement automated recovery procedures
- Use AWS Backup for centralized recovery management

## Common Issues

- symptoms: "Volume restore fails"
  diagnosis: "Snapshot corrupted, insufficient disk space, or wrong gateway type."
  resolution: "Verify snapshot status. Ensure gateway has capacity. Use correct volume type."

- symptoms: "Gateway recovery — shares/volumes not appearing"
  diagnosis: "New gateway not associated with existing resources."
  resolution: "Resources are tied to the old gateway ARN. Contact AWS Support for reassociation."

- symptoms: "Tape retrieval taking too long"
  diagnosis: "Tapes in Deep Archive have 12-hour retrieval time."
  resolution: "Expected behavior. Glacier: 3-5 hours. Deep Archive: 12 hours. Plan accordingly."

## Output Format

```yaml
root_cause: "recovery_issue — <specific_cause>"
evidence:
  - type: recovery_type
    content: "<volume restore, gateway recovery, or tape retrieval>"
  - type: source_data
    content: "<snapshot ID, gateway ARN, or tape ARN>"
  - type: recovery_status
    content: "<recovery operation status>"
severity: HIGH
mitigation:
  immediate: "Complete the recovery operation"
  long_term: "Test recovery procedures regularly and document RTO/RPO"
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
