---
title: "E2 — File Metadata Preservation Issues"
description: "Diagnose file metadata not being preserved during DataSync transfers"
status: active
severity: MEDIUM
triggers:
  - "metadata not preserved"
  - "permissions lost"
  - "timestamps wrong"
  - "ownership changed"
  - "ACLs missing"
owner: devops-agent
objective: "Identify why file metadata is not preserved and configure correct metadata handling"
context: "DataSync preserves metadata by default but behavior varies by source-destination combination. POSIX metadata (uid, gid, permissions) works for NFS/EFS/FSx for Lustre. NTFS metadata (ACLs) works for SMB/FSx for Windows. S3 stores metadata as object metadata. Not all metadata types transfer between all location types."
---

## Phase 1 — Triage

MUST:
- Check task metadata options: `aws datasync describe-task --task-arn <task-arn> --query 'Options.{Uid:Uid,Gid:Gid,PosixPermissions:PosixPermissions,PreserveDeletedFiles:PreserveDeletedFiles,Atime:Atime,Mtime:Mtime}'`
- Identify source and destination location types
- Check if the metadata type is supported for the source-destination combination
- Compare metadata at source and destination for sample files

SHOULD:
- Check if NONE is set for any metadata option (disables preservation)
- Verify NFS/SMB user mapping between source and destination
- Check EFS access point settings that may override uid/gid

MAY:
- Review DataSync documentation for metadata compatibility matrix
- Check if destination filesystem supports the metadata type

## Phase 2 — Remediate

MUST:
- Configure metadata preservation options appropriately:
  `aws datasync update-task --task-arn <task-arn> --options Uid=INT_VALUE,Gid=INT_VALUE,PosixPermissions=PRESERVE,Mtime=PRESERVE,Atime=BEST_EFFORT`
- Understand that some metadata cannot transfer between incompatible location types

SHOULD:
- For S3 destinations, understand metadata is stored as S3 object metadata
- For cross-filesystem-type transfers, document expected metadata behavior

MAY:
- Use post-transfer scripts to set metadata that DataSync cannot preserve
- Configure EFS access points to enforce specific uid/gid

## Common Issues

- symptoms: "File permissions different at destination"
  diagnosis: "PosixPermissions set to NONE or destination doesn't support POSIX."
  resolution: "Set PosixPermissions=PRESERVE. Verify destination supports POSIX metadata."

- symptoms: "Timestamps not matching after transfer"
  diagnosis: "Mtime or Atime set to NONE, or timestamp precision differs."
  resolution: "Set Mtime=PRESERVE. Note: S3 has second-level precision only."

- symptoms: "File ownership (uid/gid) changed at destination"
  diagnosis: "Uid/Gid set to NONE or NFS uid/gid mapping differs."
  resolution: "Set Uid=INT_VALUE and Gid=INT_VALUE to preserve numeric IDs."

## Output Format

```yaml
root_cause: "metadata_issue — <specific_cause>"
evidence:
  - type: metadata_options
    content: "<task metadata configuration>"
  - type: location_types
    content: "<source and destination location types>"
  - type: metadata_comparison
    content: "<metadata at source vs destination>"
severity: MEDIUM
mitigation:
  immediate: "Configure correct metadata preservation options"
  long_term: "Document metadata compatibility for your source-destination combination"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — File metadata includes ownership and permissions |
| Blast Radius | MEDIUM — Incorrect metadata preservation can break file access at destination |
| Rollback Complexity | MEDIUM — Metadata changes at destination may require manual correction |
| Compliance Scope | MEDIUM — File permissions and ownership are security controls |

## Escalation Conditions

- Escalate to security if file permissions at destination are more permissive than source
- Escalate to application owners if metadata changes break application access
- Escalate to compliance if POSIX permissions are required for regulatory compliance

## Data Sensitivity

- **File ownership (uid/gid)** reveals user and group information
- **File permissions** are security controls — incorrect settings create access risks
- **Timestamps** may be relevant for audit trails

## PROHIBITED Actions

- Do NOT set metadata preservation to NONE for security-sensitive file transfers
- Do NOT ignore permission mismatches between source and destination
- Do NOT change metadata options on active production transfer tasks without testing

## Phase 3 — Rollback

- Restore previous metadata options: `aws datasync update-task --task-arn <arn> --options Uid=INT_VALUE,Gid=INT_VALUE,PosixPermissions=PRESERVE`
- Re-run the task to correct metadata at destination
- Manually fix permissions at destination if immediate correction is needed

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
