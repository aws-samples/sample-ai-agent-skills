---
title: "C1 — Admin Container Access"
description: "Diagnose Bottlerocket admin container enablement and access issues"
status: active
severity: MEDIUM
triggers: ["cannot access admin container", "sheltie not working", "admin container disabled", "need shell access"]
owner: devops-agent
objective: "Enable and restore admin container access for Bottlerocket host-level diagnostics"
context: "The admin container is disabled by default for security. It provides sheltie for host namespace access. Enable via user data or SSM."
---

## Phase 1 — Triage

MUST:
- Check admin container status: `aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="apiclient get settings.host-containers.admin"`
- Verify SSM connectivity to the control container: `aws ssm describe-instance-information --filters "Key=InstanceIds,Values=<instance-id>"`
- Check if admin container is enabled in user data or launch template

SHOULD:
- Verify the instance has the SSM IAM permissions: `aws iam list-attached-role-policies --role-name <instance-role>`
- Check if the admin container was previously enabled and then disabled

MAY:
- Review CloudTrail for SSM session history: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm.amazonaws.com --max-results 10`

## Phase 2 — Remediate

MUST:
- Enable admin container via control container: `apiclient set host-containers.admin.enabled=true`
- Commit and verify: `apiclient get settings.host-containers.admin`
- Access host namespace: `enter-admin-container` then `sheltie`

SHOULD:
- Configure SSH key for admin container if SSH access is needed: `apiclient set host-containers.admin.user-data="<base64-encoded-ssh-key>"`
- Document the reason for enabling admin container access

MAY:
- Set up admin container enablement in user data for future instances

## Safety Ratings

```
safety_ratings:
  - "apiclient get settings.host-containers.admin: GREEN — read-only status check"
  - "apiclient set host-containers.admin.enabled=true: YELLOW — enables privileged access"
  - "enter-admin-container / sheltie: RED — provides full host namespace access"
  - "ssm describe-instance-information: GREEN — read-only SSM status check"
```

## Escalation Conditions

- Control container is also unreachable (no SSM access at all)
- Admin container enabled but sheltie fails with permission errors
- Instance requires host-level debugging but security policy prohibits admin container
- Admin container enablement causes SELinux denials

## Data Sensitivity

- `apiclient get settings.host-containers.admin`: LOW — returns enabled/disabled status
- `enter-admin-container` / `sheltie`: HIGH — provides full host filesystem and process access
- SSH key configuration: HIGH — contains authentication credentials
- Host namespace access: HIGH — exposes all system processes and data

## Prohibited Actions

- NEVER leave the admin container enabled in production after debugging is complete
- NEVER share SSH keys configured for the admin container in logs or tickets
- NEVER use the admin container to modify the root filesystem directly
- NEVER suggest enabling admin container as a permanent configuration in production

## Phase 3 — Rollback

1. Disable admin container after debugging: `apiclient set host-containers.admin.enabled=false`
2. Verify admin container is stopped: `apiclient get settings.host-containers.admin`
3. Remove SSH keys if they were configured: `apiclient set host-containers.admin.user-data=""`
4. Audit any changes made via sheltie and revert if necessary
5. Document what was accessed and changed during the admin session

## Common Issues

- symptoms: "apiclient set host-containers.admin.enabled=true returns success but admin container does not start"
  diagnosis: "The admin container image may not be cached or the container runtime has an issue."
  resolution: "Wait 30 seconds for the container to pull and start. If still failing, reboot: apiclient reboot."

- symptoms: "SSM session fails with TargetNotConnected"
  diagnosis: "Control container is not running or instance lacks SSM IAM permissions."
  resolution: "Verify IAM instance profile includes AmazonSSMManagedInstanceCore. Check VPC endpoints for SSM."

## Output Format

```yaml
root_cause: "admin_container — <specific_cause>"
evidence:
  - type: access_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Enable admin container for debugging"
  long_term: "Establish secure admin access procedures"
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
