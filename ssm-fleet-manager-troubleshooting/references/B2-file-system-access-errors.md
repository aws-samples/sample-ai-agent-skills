---
title: "B2 — File System Access Errors"
description: "Diagnose Fleet Manager file system browsing and access errors"
status: active
severity: MEDIUM
triggers: ["file system not loading", "cannot browse files", "file access denied in Fleet Manager"]
owner: devops-agent
objective: "Resolve Fleet Manager file system browsing issues"
context: "Fleet Manager file system browsing uses Session Manager under the hood. It requires SSM Agent 2.3.68+, proper IAM permissions, and Session Manager prerequisites."
---

## Phase 1 — Triage

MUST:
- Verify SSM Agent version supports file system: `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<instance-id> --query 'InstanceInformationList[*].{Id:InstanceId,Agent:AgentVersion}'`
- Check connection status: `aws ssm get-connection-status --target <instance-id>`
- Verify IAM permissions include `ssm:StartSession`

SHOULD:
- Check if the OS user running SSM Agent has file system read permissions
- Verify Session Manager is functional independently of Fleet Manager

MAY:
- Test file access via direct Session Manager session
- Review SSM Agent logs for file operation errors

## Phase 2 — Remediate

MUST:
- Update SSM Agent if below 2.3.68
- Ensure IAM policy allows `ssm:StartSession` for the instance
- Verify the SSM Agent service account has read access to target directories

SHOULD:
- Check OS-level file permissions (Linux: check ssm-user permissions; Windows: check Local System account)
- Verify disk space is not exhausted on the target instance

MAY:
- Configure Session Manager to run as a specific OS user with broader file access

## Common Issues

- symptoms: "File system tab shows 'Unable to load file system' error"
  diagnosis: "SSM Agent version too old or Session Manager prerequisites not met."
  resolution: "Update SSM Agent to latest version. Verify Session Manager connectivity."

- symptoms: "Can browse some directories but get 'Access Denied' on others"
  diagnosis: "OS-level file permissions restrict the SSM Agent service account."
  resolution: "Check permissions for the ssm-user (Linux) or Local System (Windows) account."

## Safety Ratings

```
safety_ratings:
  - "describe-instance-information: GREEN — read-only query"
  - "get-connection-status: GREEN — read-only check"
  - "ssm:StartSession for file browsing: YELLOW — reads file system contents"
  - "send-command to check permissions: GREEN — read-only diagnostic"
```

## Escalation Conditions

- File system access fails on all instances in the account simultaneously
- Access errors occur for system-critical directories needed for troubleshooting
- File system browsing causes SSM Agent to crash or become unresponsive
- Compliance requirements mandate specific file access audit logging

## Data Sensitivity

- `aws ssm describe-instance-information`: LOW — instance metadata only
- File system browsing: HIGH — may expose sensitive file contents, configuration files, credentials
- `aws ssm start-session`: HIGH — provides file system access to the instance
- SSM Agent logs: MEDIUM — may contain file paths and error details

## Prohibited Actions

- NEVER modify file permissions broadly (chmod 777) to fix access issues
- NEVER browse file systems on production instances without authorization
- NEVER download sensitive files (credentials, keys, certificates) through Fleet Manager
- NEVER change the SSM Agent service account to root/Administrator without security review

## Phase 3 — Rollback

1. If file permissions were changed, restore original permissions from backup
2. If SSM Agent was updated and file browsing broke, downgrade agent version
3. If Session Manager configuration was modified, restore previous settings
4. Verify file system browsing returns to expected behavior
5. Document any file permission changes made during troubleshooting

## Output Format

```yaml
root_cause: "file_system_access — <specific_cause>"
evidence:
  - type: access_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Restore file system browsing"
  long_term: "Standardize SSM Agent and OS permissions"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
