---
title: "G1 — Service Integration Issues"
description: "Diagnose AWS service integration with Organizations problems"
status: active
severity: MEDIUM
triggers:
  - "service integration"
  - "service not working with org"
  - "organization service"
  - "cross-account service"
owner: devops-agent
objective: "Configure AWS service integration with Organizations correctly"
context: "Many AWS services integrate with Organizations for cross-account management (CloudTrail, Config, GuardDuty, Security Hub, etc.). Integration requires enabling trusted access, configuring delegated administrators, and ensuring SCPs don't block service operations."
---

## Phase 1 — Triage

MUST:
- List enabled service integrations: `aws organizations list-aws-service-access-for-organization`
- Check if the specific service has trusted access enabled
- Verify delegated admin is registered if applicable: `aws organizations list-delegated-administrators --service-principal <service>.amazonaws.com`
- Check SCPs for statements that may block the service

SHOULD:
- Verify service-linked roles exist in member accounts
- Check CloudTrail for service integration events
- Verify the service is available in the target region

MAY:
- Check service-specific configuration in member accounts
- Review service documentation for Organizations integration requirements

## Phase 2 — Remediate

MUST:
- Enable trusted access for the service
- Register delegated administrator if needed
- Ensure SCPs allow the service's required actions

SHOULD:
- Follow the service's recommended setup for Organizations integration
- Document which services are integrated and their configuration
- Test integration in a sandbox account first

MAY:
- Use AWS Control Tower for automated service integration
- Implement monitoring for service integration health

## Common Issues

- symptoms: "Service cannot discover member accounts"
  diagnosis: "Trusted access is not enabled for the service."
  resolution: "Enable trusted access from the management account."

- symptoms: "Service works in management account but not member accounts"
  diagnosis: "SCPs may be blocking the service's required actions in member accounts."
  resolution: "Review SCPs and ensure they allow the service's required actions."

- symptoms: "Delegated admin cannot manage the service across accounts"
  diagnosis: "Delegated admin registration is incomplete or the service doesn't support delegation."
  resolution: "Verify delegation is registered and the service supports it."

## Output Format

```yaml
root_cause: "service_integration_issue — <specific_cause>"
evidence:
  - type: service_access
    content: "<enabled service integrations>"
  - type: delegated_admin
    content: "<delegated admin status>"
  - type: scps
    content: "<relevant SCP statements>"
severity: MEDIUM
mitigation:
  immediate: "Enable trusted access and configure delegation"
  long_term: "Document service integrations and monitor health"
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
