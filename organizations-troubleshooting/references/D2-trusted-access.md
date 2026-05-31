---
title: "D2 — Trusted Access Issues"
description: "Diagnose trusted service access configuration problems"
status: active
severity: MEDIUM
triggers:
  - "trusted access"
  - "service access"
  - "enable trusted access"
  - "service linked role"
owner: devops-agent
objective: "Configure trusted access for AWS services to work with Organizations"
context: "Trusted access allows AWS services (CloudTrail, Config, GuardDuty, etc.) to perform operations across the organization. It must be enabled per service from the management account. Enabling trusted access creates service-linked roles in member accounts."
---

## Phase 1 — Triage

MUST:
- List enabled service access: `aws organizations list-aws-service-access-for-organization`
- Check if the specific service has trusted access enabled
- Verify the caller is in the management account
- Check for service-linked roles in member accounts

SHOULD:
- Check CloudTrail for trusted access events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=EnableAWSServiceAccess --max-results 10`
- Verify the service supports trusted access

MAY:
- Check for SCPs that may block service-linked role creation in member accounts

## Phase 2 — Remediate

MUST:
- Enable trusted access from the management account: `aws organizations enable-aws-service-access --service-principal <service>.amazonaws.com`
- Verify the service appears in the enabled list
- Ensure SCPs do not block the service's operations in member accounts

SHOULD:
- Document which services have trusted access enabled
- Review trusted access periodically and disable unused services

MAY:
- Use AWS Control Tower for automated trusted access management
- Set up monitoring for trusted access changes

## Common Issues

- symptoms: "Service cannot access member accounts"
  diagnosis: "Trusted access is not enabled for the service."
  resolution: "Enable trusted access: `aws organizations enable-aws-service-access --service-principal <service>.amazonaws.com`"

- symptoms: "Cannot enable trusted access — not in management account"
  diagnosis: "Trusted access can only be enabled from the management account."
  resolution: "Switch to the management account to enable trusted access."

## Output Format

```yaml
root_cause: "trusted_access_issue — <specific_cause>"
evidence:
  - type: service_access
    content: "<enabled service access list>"
  - type: service_principal
    content: "<service principal and status>"
severity: MEDIUM
mitigation:
  immediate: "Enable trusted access for the required service"
  long_term: "Document and review trusted access configuration"
```


## Safety Ratings
- GREEN: describe-organization, list-accounts, list-policies
- YELLOW: attach-policy, create-organizational-unit, move-account
- RED: close-account, delete-policy, remove-account-from-organization

## Escalation Conditions
- "Remediation involves modifying SCPs in production"
- "Fix requires moving accounts between OUs"
- "Change affects the management account"
- "SCP modification could lock out member accounts"

## Rollback
- Pre-change: "Document current SCP attachments and OU structure"
- Verification: "Test affected account access after SCP change"
- Revert: "Detach modified SCP and reattach original"

## Data Sensitivity
- HIGH: "list-accounts reveals all account IDs, names, and emails"
- HIGH: "SCP content reveals security boundaries"
- MEDIUM: "OU structure reveals organizational hierarchy"

## Prohibited Actions
- NEVER suggest removing all SCPs from an OU
- NEVER suggest closing an account without confirming data backup
- NEVER suggest modifying the management account's policies
- NEVER suggest removing an account from the organization without root email access confirmation

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
  - command: "list-accounts"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-organization"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-policies"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest removing SCPs without understanding blast radius"
  - "NEVER suggest disabling AI opt-out policies without review"
  - "NEVER modify organization structure without approval"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
