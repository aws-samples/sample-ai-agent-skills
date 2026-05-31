---
title: "B2 — vCenter Authentication"
description: "Diagnose vCenter authentication and credential issues"
status: active
severity: HIGH
triggers: ["authentication failed", "invalid credentials", "vCenter login error", "permission denied vCenter"]
owner: devops-agent
objective: "Resolve vCenter authentication issues for Backup Gateway"
context: "Gateway stores encrypted vCenter credentials. Password rotations break the connection silently."
---

## Phase 1 — Triage

MUST:
- Check hypervisor connection state: `aws backup-gateway get-hypervisor --hypervisor-arn <hypervisor-arn>`
- Verify the vCenter user account is not locked or disabled
- Confirm the vCenter password has not been rotated since last successful connection

SHOULD:
- Verify the vCenter user has required permissions: ReadOnly + VirtualMachine.State.CreateSnapshot + VirtualMachine.Provisioning.DiskAccess
- Check vCenter audit logs for failed login attempts

MAY:
- Test vCenter credentials independently (e.g., via PowerCLI or vSphere client)
- Review vCenter password policy for expiration settings

## Phase 2 — Remediate

MUST:
- Update stored credentials: `aws backup-gateway update-hypervisor --hypervisor-arn <arn> --username <user> --password <new-pass>`
- Verify the vCenter user account is active and not locked
- Confirm the user has required VMware permissions

SHOULD:
- Use a dedicated service account with non-expiring password (or managed rotation)
- Document the required vCenter permissions for the service account

MAY:
- Set up alerts for vCenter password expiration
- Implement automated credential rotation with Secrets Manager

## Safety Ratings

```
safety_ratings:
  - "backup-gateway get-hypervisor: GREEN — read-only hypervisor details"
  - "backup-gateway update-hypervisor: YELLOW — updates stored credentials"
  - "vCenter login testing: GREEN — read-only authentication test"
  - "vCenter permission changes: RED — modifies VMware access control"
```

## Escalation Conditions

- vCenter admin account is required to unlock or reset the service account
- vCenter password policy enforces rotation that conflicts with gateway operations
- Required VMware permissions need vCenter administrator approval
- Multiple hypervisor connections fail due to centralized authentication changes

## Data Sensitivity

- `aws backup-gateway get-hypervisor`: MEDIUM — reveals vCenter host and username
- vCenter credentials: CRITICAL — provides access to the VMware environment
- vCenter audit logs: MEDIUM — reveals authentication patterns
- VMware permission assignments: MEDIUM — reveals access control configuration

## Prohibited Actions

- NEVER log or display vCenter passwords in troubleshooting output
- NEVER use the vCenter administrator@vsphere.local account for Backup Gateway
- NEVER grant more VMware permissions than required for backup operations
- NEVER disable vCenter authentication requirements to fix connection issues

## Phase 3 — Rollback

1. If credentials were updated incorrectly, update again with the correct password
2. If the vCenter user was modified, restore original permissions
3. Verify hypervisor connection is re-established after credential update
4. Check that VM discovery and backup operations resume
5. Document the credential update and notify the VMware admin team

## Common Issues

- symptoms: "Backup jobs fail with authentication error after vCenter password rotation"
  diagnosis: "Stored credentials on the gateway are stale after password change."
  resolution: "Update credentials: aws backup-gateway update-hypervisor --hypervisor-arn <arn> --password <new-password>"

## Output Format

```yaml
root_cause: "vcenter_authentication — <specific_cause>"
evidence:
  - type: auth_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Update vCenter credentials"
  long_term: "Implement credential rotation alerts"
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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
