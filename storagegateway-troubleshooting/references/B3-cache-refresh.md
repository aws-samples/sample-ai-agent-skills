---
title: "B3 — Cache Refresh Issues"
description: "Diagnose file share cache refresh problems when S3 objects change outside the gateway"
status: active
severity: MEDIUM
triggers:
  - "cache refresh"
  - "stale files"
  - "S3 changes not visible"
  - "files out of date"
owner: devops-agent
objective: "Identify and resolve cache refresh issues for file gateway shares"
context: "When S3 objects are modified directly (not through the gateway), the file share cache must be refreshed to reflect changes. Cache refresh scans the S3 bucket/prefix and updates the local cache. Large buckets can take significant time to refresh. Refresh can be triggered manually, via API, or on a schedule."
---

## Phase 1 — Triage

MUST:
- Check if files were modified directly in S3 (bypassing gateway)
- Trigger cache refresh: `aws storagegateway refresh-cache --file-share-arn <file-share-arn>`
- Check refresh status via CloudWatch Logs or notifications
- Verify the file share's S3 bucket and prefix are correct

SHOULD:
- Check if automated cache refresh is configured on the file share
- Monitor refresh duration for large buckets
- Verify S3 event notifications are configured (for automated refresh)

MAY:
- Check if S3 versioning is causing unexpected behavior
- Review file share notification policy configuration

## Phase 2 — Remediate

MUST:
- Trigger manual refresh for immediate needs: `aws storagegateway refresh-cache --file-share-arn <arn> --folder-list "/path/to/changed/folder/"`
- Use folder-list parameter to refresh specific paths (faster than full refresh)

SHOULD:
- Configure automated cache refresh schedule on the file share
- Set up S3 event notifications to trigger Lambda-based cache refresh
- Use folder-specific refresh for large buckets

MAY:
- Implement a workflow that triggers refresh after S3 uploads
- Monitor refresh completion with CloudWatch Events

## Common Issues

- symptoms: "New files uploaded to S3 not visible on file share"
  diagnosis: "Cache not refreshed after direct S3 upload."
  resolution: "Trigger cache refresh. Consider automated refresh schedule."

- symptoms: "Cache refresh takes very long"
  diagnosis: "Large number of objects in S3 bucket/prefix."
  resolution: "Use folder-list to refresh specific paths. Reduce prefix scope."

- symptoms: "Deleted S3 objects still visible on file share"
  diagnosis: "Cache not refreshed after S3 object deletion."
  resolution: "Trigger cache refresh. Deleted objects will be removed from cache."

## Output Format

```yaml
root_cause: "cache_refresh — <specific_cause>"
evidence:
  - type: s3_changes
    content: "<what changed in S3>"
  - type: refresh_status
    content: "<cache refresh result>"
  - type: share_config
    content: "<file share S3 bucket and prefix>"
severity: MEDIUM
mitigation:
  immediate: "Trigger cache refresh for affected paths"
  long_term: "Configure automated cache refresh or S3 event-driven refresh"
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
