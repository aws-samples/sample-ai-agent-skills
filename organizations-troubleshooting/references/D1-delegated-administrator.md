---
title: "D1 — Delegated Administrator Issues"
description: "Diagnose delegated administrator registration and operation problems"
status: active
severity: MEDIUM
triggers:
  - "delegated administrator"
  - "delegated admin"
  - "register delegated"
  - "deregister delegated"
owner: devops-agent
objective: "Configure delegated administration correctly for AWS services"
context: "Delegated administration allows a member account to perform organizational operations for a specific service. Not all services support delegation. The management account must register the delegated admin. Each service has its own delegation model and limits (typically max 5 delegated admins per service)."
---

## Phase 1 — Triage

MUST:
- List delegated administrators: `aws organizations list-delegated-administrators`
- List delegated services for an account: `aws organizations list-delegated-services-for-account --account-id <account-id>`
- Verify the service supports delegated administration
- Check if the caller is in the management account

SHOULD:
- Check the maximum delegated admin limit for the service
- Verify the member account is active and in the organization
- Check CloudTrail for delegation events

MAY:
- Check service-specific documentation for delegation requirements
- Verify the delegated admin has the required IAM permissions

## Phase 2 — Remediate

MUST:
- Register the delegated admin from the management account: `aws organizations register-delegated-administrator --account-id <account-id> --service-principal <service>.amazonaws.com`
- Verify the registration was successful
- Ensure the delegated admin account has appropriate IAM permissions

SHOULD:
- Document which accounts are delegated admins for which services
- Limit delegated admin to trusted accounts only
- Review delegated admin registrations periodically

MAY:
- Use separate accounts for different delegated admin responsibilities
- Implement automation for delegated admin lifecycle management

## Common Issues

- symptoms: "Cannot register delegated admin — service not supported"
  diagnosis: "The AWS service does not support delegated administration."
  resolution: "Check service documentation. Not all services support delegation."

- symptoms: "Maximum delegated administrators reached"
  diagnosis: "The service has reached its maximum delegated admin limit (typically 5)."
  resolution: "Deregister an existing delegated admin before registering a new one."

- symptoms: "Delegated admin cannot perform organizational operations"
  diagnosis: "The account is registered but lacks IAM permissions for the service."
  resolution: "Add the required IAM permissions to the delegated admin account."

## Output Format

```yaml
root_cause: "delegated_admin_issue — <specific_cause>"
evidence:
  - type: delegated_admins
    content: "<registered delegated administrators>"
  - type: delegated_services
    content: "<services delegated to the account>"
severity: MEDIUM
mitigation:
  immediate: "Register or fix delegated admin configuration"
  long_term: "Document delegation strategy and review periodically"
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
