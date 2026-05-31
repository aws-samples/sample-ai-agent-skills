---
title: "D1 — Cross-Account Sharing"
description: "Diagnose Lake Formation cross-account sharing issues"
status: active
severity: HIGH
triggers:
  - "cross-account sharing failed"
  - "cannot share table cross-account"
  - "external account access"
  - "cross-account grant error"
owner: devops-agent
objective: "Identify and resolve LF cross-account sharing issues"
context: "LF cross-account sharing grants permissions to external accounts. This creates RAM resource shares. The recipient must accept the RAM invitation and create a resource link in their catalog. Both accounts need proper LF configuration."
---

## Phase 1 — Triage

MUST:
- Check cross-account grants: `aws lakeformation list-permissions --principal '{"DataLakePrincipalIdentifier":"<external-account-id>"}'`
- Verify RAM share was created: `aws ram get-resource-shares --resource-owner SELF`
- Check LF data lake settings allow cross-account: `aws lakeformation get-data-lake-settings`
- Verify the recipient accepted the RAM invitation

SHOULD:
- Check if the recipient created a resource link
- Verify the data location is registered in the granting account
- Check IAMAllowedPrincipals status on the shared resource
- Verify both accounts have LF admins configured

MAY:
- Check CloudTrail in both accounts for sharing events
- Verify organization sharing is enabled (if applicable)

## Phase 2 — Remediate

MUST:
- Grant cross-account LF permissions from the owner account
- Ensure recipient accepts RAM invitation
- Recipient creates resource link: `aws glue create-database` with target database

SHOULD:
- Configure LF admins in both accounts
- Register data locations in the owner account
- Grant appropriate permissions (SELECT, DESCRIBE) to the external account

MAY:
- Use LF-Tags for cross-account sharing at scale
- Set up monitoring for cross-account access

## Common Issues

- symptoms: "Cross-account grant succeeds but recipient can't access"
  diagnosis: "RAM invitation not accepted or resource link not created."
  resolution: "Accept RAM invitation. Create resource link in recipient catalog."

- symptoms: "Cannot grant to external account"
  diagnosis: "LF settings don't allow cross-account grants."
  resolution: "Update data lake settings to allow external account grants."

## Output Format

```yaml
root_cause: "cross_account_sharing — <specific_cause>"
evidence:
  - type: lf_grants
    content: "<cross-account permission grants>"
  - type: ram_shares
    content: "<RAM resource share status>"
severity: HIGH
mitigation:
  immediate: "Fix cross-account sharing configuration"
  long_term: "Standardize cross-account sharing patterns"
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
