---
title: "E1 — Logical Directory Mapping Issues"
description: "Diagnose logical directory mapping configuration and access problems"
status: active
severity: MEDIUM
triggers:
  - "directory mapping"
  - "logical directory"
  - "virtual directory"
  - "chroot"
  - "Entry Target"
  - "cannot see files"
  - "wrong directory"
owner: devops-agent
objective: "Resolve logical directory mapping configuration issues that prevent users from accessing expected paths"
context: "Logical directory mappings (HomeDirectoryType=LOGICAL) create a virtual filesystem by mapping Entry paths (what the user sees) to Target paths (actual S3 bucket/prefix locations). Users are chroot-jailed to their mapped directories. At least one mapping with Entry '/' is required. Logical mappings are only supported with S3 backends."
---

## Phase 1 — Triage

MUST:
- Check user's directory type and mappings: `aws transfer describe-user --server-id <server-id> --user-name <username> --query 'User.{HomeDirectoryType:HomeDirectoryType,HomeDirectoryMappings:HomeDirectoryMappings,HomeDirectory:HomeDirectory}'`
- Verify HomeDirectoryType is LOGICAL (not PATH)
- Verify at least one mapping has Entry "/" as the root
- Check that Target paths point to valid S3 bucket/prefix locations: `aws s3 ls s3://<target-path>/`
- Verify the IAM role has access to all Target S3 paths

SHOULD:
- Check if session policy restricts access to any of the Target paths
- Verify Target paths use the correct format: /<bucket-name>/<prefix>
- Check for overlapping or conflicting Entry paths
- Test with the Transfer Family console's user test feature

MAY:
- Check if the custom IdP returns HomeDirectoryDetails correctly (JSON string format)
- Verify no trailing slashes cause path resolution issues

## Phase 2 — Remediate

MUST:
- Ensure at least one mapping has Entry "/" pointing to the user's root Target
- Use correct Target format: /<bucket-name>/optional-prefix (leading slash, no trailing slash)
- Ensure the IAM role and session policy allow access to all mapped Target paths
- For custom IdP, return HomeDirectoryDetails as a JSON string: `"[{\"Entry\":\"/\",\"Target\":\"/<bucket>/<prefix>\"}]"`

SHOULD:
- Keep mappings simple — avoid deeply nested virtual directory structures
- Use ${transfer:UserName} in Target paths for per-user isolation
- Document the mapping structure for operational reference

MAY:
- Implement dynamic mappings via custom IdP based on user attributes
- Use multiple mappings to give users access to shared directories alongside their home

## Common Issues

- symptoms: "User sees empty directory after login"
  diagnosis: "Target path does not exist in S3 or IAM role lacks s3:ListBucket permission on the bucket."
  resolution: "Verify Target path exists. Add s3:ListBucket with the bucket ARN (not object ARN) to the role."

- symptoms: "Logical mapping returns error on login"
  diagnosis: "No mapping with Entry '/' defined, or HomeDirectoryType is not set to LOGICAL."
  resolution: "Add a mapping with Entry '/' and set HomeDirectoryType to LOGICAL."

- symptoms: "User can access one mapped directory but not another"
  diagnosis: "Session policy or IAM role restricts access to specific S3 prefixes, not covering all Target paths."
  resolution: "Update the session policy to include all Target paths in the S3 resource ARNs."

## Output Format

```yaml
root_cause: "directory_mapping — <specific_cause>"
evidence:
  - type: mappings
    content: "<Entry/Target mapping configuration>"
  - type: s3_access
    content: "<S3 path existence and permissions>"
  - type: session_policy
    content: "<session policy restrictions>"
severity: MEDIUM
mitigation:
  immediate: "Fix mapping configuration or IAM permissions"
  long_term: "Standardize mapping patterns and implement dynamic mappings via custom IdP"
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

- Transferred files may contain any type of sensitive data
- SSH keys and credentials enable server access
- Directory mappings reveal S3/EFS data organization
- Classification: CONFIDENTIAL - file transfer content and access credentials

## Prohibited Actions

- NEVER modify production server endpoints during active file transfers
- NEVER change authentication providers without user notification
- NEVER delete SSH host keys without coordinating with all SFTP clients
- NEVER modify directory mappings that could expose unauthorized S3/EFS paths

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
  - "NEVER suggest disabling SFTP encryption"
  - "NEVER suggest FTP protocol for production"
  - "NEVER suggest overly broad IAM roles for transfer users"
