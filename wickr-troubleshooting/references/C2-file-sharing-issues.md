---
title: "C2 — File Sharing Issues"
description: "Diagnose file sharing and attachment failures"
status: active
severity: MEDIUM
triggers: ["file upload failed", "cannot share file", "attachment error", "file too large"]
owner: devops-agent
objective: "Resolve file sharing and attachment issues on the Wickr network"
context: "File sharing limits are configured at the network level. Files are encrypted end-to-end like messages."
---

## Phase 1 — Triage

MUST:
- Verify network is active: `aws wickr get-network --network-id <id>`
- Check network-level file sharing settings in admin console
- Review CloudWatch metrics for file transfer errors: `aws cloudwatch get-metric-statistics --namespace AWS/Wickr --metric-name FileTransferFailures --dimensions Name=NetworkId,Value=<id> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Verify file size against network maximum file size limit
- Check client network connectivity and bandwidth
- Review if the issue affects all users or specific users

MAY:
- Test file sharing with a small test file
- Check client application version for known file sharing bugs

## Phase 2 — Remediate

MUST:
- Verify file size is within network-configured limits
- Ensure sender and recipient have file sharing permissions
- Confirm network connectivity supports file transfer bandwidth

SHOULD:
- Increase file size limit in admin console if business requires larger files
- Update client applications to latest version

MAY:
- Implement file compression guidance for users
- Configure file type restrictions if security policy requires

## Safety Ratings

```
safety_ratings:
  - "wickr get-network: GREEN — read-only network details"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "admin console file settings: YELLOW — modifies network-level file policies"
  - "client update: GREEN — non-destructive application update"
```

## Escalation Conditions

- File sharing completely broken for all users
- Files corrupted during transfer
- File sharing works for some file types but not others
- Network-level file size limit cannot be increased

## Data Sensitivity

- `aws wickr get-network`: LOW — returns network metadata
- File sharing metrics: LOW — aggregate transfer statistics
- File content: NOT ACCESSIBLE — end-to-end encrypted
- File metadata: MEDIUM — reveals file names and sizes

## Prohibited Actions

- NEVER attempt to access shared file content
- NEVER disable file sharing encryption
- NEVER set file size limits to unlimited without security review
- NEVER allow unrestricted file types without security policy approval

## Phase 3 — Rollback

1. If file sharing settings were modified, restore original limits
2. If file type restrictions were changed, restore original restrictions
3. Verify file sharing works correctly after rollback
4. Confirm no unauthorized file types were shared during changes
5. Document file sharing configuration changes

## Common Issues

- symptoms: "File upload fails with 'File too large' error"
  diagnosis: "File exceeds the network-configured maximum file size."
  resolution: "Check network file size limit in admin console. Increase if business requires or compress the file."

- symptoms: "File sharing works on desktop but not mobile"
  diagnosis: "Mobile client may have different file handling or outdated version."
  resolution: "Update mobile client to latest version. Check mobile network connectivity."

## Output Format

```yaml
root_cause: "file_sharing — <specific_cause>"
evidence:
  - type: file_transfer_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Resolve file sharing issue"
  long_term: "Configure appropriate file sharing policies"
```

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
