---
title: "D2 — RAM Integration Issues"
description: "Diagnose Lake Formation and RAM integration problems"
status: active
severity: HIGH
triggers:
  - "RAM integration error"
  - "resource share not created"
  - "RAM invitation for LF"
  - "LF RAM share"
owner: devops-agent
objective: "Identify and resolve LF-RAM integration issues"
context: "Lake Formation uses RAM for cross-account sharing. LF grants to external accounts automatically create RAM resource shares. Issues can arise from RAM not being enabled, invitation acceptance failures, or organization sharing configuration."
---

## Phase 1 — Triage

MUST:
- Check RAM resource shares: `aws ram get-resource-shares --resource-owner SELF --resource-type glue:Table`
- Verify RAM invitations: `aws ram get-resource-share-invitations`
- Check if organization sharing is enabled in RAM
- Verify RAM service-linked role exists

SHOULD:
- Check CloudTrail for RAM events related to LF
- Verify the RAM share status is ACTIVE
- Check principal associations in the RAM share
- Verify the Glue resource is in the RAM share

MAY:
- Check for SCPs blocking RAM operations
- Verify cross-account IAM permissions

## Phase 2 — Remediate

MUST:
- Ensure RAM is properly configured for LF integration
- Accept pending RAM invitations in recipient account
- Verify RAM share is ACTIVE with correct resources

SHOULD:
- Enable organization sharing in RAM for org-wide LF sharing
- Verify RAM permissions allow LF resource sharing
- Test cross-account access after RAM setup

MAY:
- Automate RAM invitation acceptance
- Set up monitoring for RAM share status

## Common Issues

- symptoms: "LF cross-account grant doesn't create RAM share"
  diagnosis: "RAM service issue or permission problem."
  resolution: "Check RAM service-linked role. Verify IAM permissions for RAM operations."

- symptoms: "RAM invitation pending but cannot accept"
  diagnosis: "Recipient lacks ram:AcceptResourceShareInvitation permission."
  resolution: "Add RAM acceptance permissions in the recipient account."

## Output Format

```yaml
root_cause: "ram_integration — <specific_cause>"
evidence:
  - type: ram_shares
    content: "<RAM share status and resources>"
  - type: invitations
    content: "<RAM invitation status>"
severity: HIGH
mitigation:
  immediate: "Fix RAM integration issue"
  long_term: "Enable organization sharing and automate RAM management"
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
