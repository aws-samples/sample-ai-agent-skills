---
title: "A2 — Permission Revoke Issues"
description: "Diagnose Lake Formation permission revoke failures"
status: active
severity: HIGH
triggers:
  - "revoke failed"
  - "cannot revoke permissions"
  - "RevokePermissions error"
  - "permission still active after revoke"
owner: devops-agent
objective: "Identify and resolve LF permission revoke issues"
context: "Permission revokes can fail if the caller lacks admin privileges, the permission doesn't exist as specified, or the revoke parameters don't exactly match the grant. Revokes must match the original grant's resource and permission specification exactly."
---

## Phase 1 — Triage

MUST:
- List current permissions: `aws lakeformation list-permissions --principal '{"DataLakePrincipalIdentifier":"<principal-arn>"}'`
- Verify caller is LF admin: `aws lakeformation get-data-lake-settings --query 'DataLakeSettings.DataLakeAdmins'`
- Check the exact permission specification to revoke
- Verify the revoke parameters match the original grant exactly

SHOULD:
- Check CloudTrail for revoke attempts
- Verify no cached credentials are maintaining access
- Check if the permission was granted with grant option
- Verify the resource specification matches exactly

MAY:
- Check for permissions granted through LF-Tags (cannot be revoked as named resource grants)
- Verify no other grants provide equivalent access

## Phase 2 — Remediate

MUST:
- Match revoke parameters exactly to the original grant
- Use LF admin credentials for revoke operations
- Verify the permission is actually removed after revoke

SHOULD:
- List all permissions for the principal to find the exact grant
- Revoke both the permission and grant option if applicable
- Test access after revoke to confirm removal

MAY:
- Audit all permissions periodically
- Automate permission lifecycle management

## Common Issues

- symptoms: "Revoke succeeds but access still works"
  diagnosis: "Another grant (possibly via LF-Tags) provides equivalent access."
  resolution: "List ALL permissions for the principal. Check LF-Tag-based grants."

- symptoms: "Revoke fails with 'permission not found'"
  diagnosis: "Revoke parameters don't match the original grant exactly."
  resolution: "List permissions to find exact grant specification. Match parameters exactly."

## Output Format

```yaml
root_cause: "revoke_issue — <specific_cause>"
evidence:
  - type: current_permissions
    content: "<current permissions for the principal>"
  - type: revoke_attempt
    content: "<revoke parameters and result>"
severity: HIGH
mitigation:
  immediate: "Fix revoke parameters or identify alternative grants"
  long_term: "Implement permission auditing and lifecycle management"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** revoke Lake Formation permissions on production databases without explicit customer approval
- **NEVER** modify IAMAllowedPrincipals settings without understanding the full permission inheritance chain
- **NEVER** deregister S3 data locations that are actively referenced by Glue tables or crawlers
- **NEVER** disable hybrid access mode on databases with active IAM-based consumers
- **NEVER** alter cross-account resource shares (RAM) without coordinating with all participating accounts

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
