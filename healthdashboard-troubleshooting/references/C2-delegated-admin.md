---
title: "C2 — Delegated Admin"
description: "Diagnose delegated administrator setup for Health"
status: active
severity: MEDIUM
triggers:
  - "delegated admin health"
  - "health admin delegation"
  - "non-management account health"
  - "delegate health access"
owner: devops-agent
objective: "Resolve delegated administrator configuration for Health"
context: "Delegated administration allows a member account to access organizational health data without using the management account. Requires Organizations with all features and Health enabled as trusted service."
---

## Phase 1 — Triage

MUST:
- Check current delegated administrators: `aws organizations list-delegated-administrators --service-principal health.amazonaws.com`
- Verify Organizations has all features: `aws organizations describe-organization --query 'Organization.FeatureSet'`
- Check Health is enabled as trusted service: `aws organizations list-aws-service-access-for-organization`
- Verify the delegated admin account has appropriate IAM permissions

SHOULD:
- Verify the delegated admin account has Business/Enterprise Support
- Check if the account is an active member of the organization
- Review IAM policies for health:Describe*ForOrganization permissions

MAY:
- Check if multiple delegated admins are configured
- Review CloudTrail for delegation API events

## Phase 2 — Remediate

MUST:
- Enable Health as trusted service first
- Register delegated admin: `aws organizations register-delegated-administrator --account-id <id> --service-principal health.amazonaws.com`
- Grant IAM permissions for organizational health API in the delegated account

SHOULD:
- Use a dedicated operations account as delegated admin
- Document delegated admin configuration
- Test organizational health access from delegated account

MAY:
- Set up automated health event processing in delegated admin account
- Implement cross-account EventBridge rules from delegated admin

## Common Issues

- symptoms: "Cannot register delegated administrator"
  diagnosis: "Health not enabled as trusted service or Organizations not using all features."
  resolution: "Enable all features and Health service access first."

- symptoms: "Delegated admin cannot see org health events"
  diagnosis: "IAM permissions missing or support plan insufficient."
  resolution: "Add health:Describe*ForOrganization permissions. Verify Business/Enterprise Support."

- symptoms: "Delegated admin registration fails"
  diagnosis: "Account is the management account (cannot be delegated admin) or already registered."
  resolution: "Use a member account. Check existing delegated admins."

## Output Format

```yaml
root_cause: "delegated_admin — <specific_cause>"
evidence:
  - type: delegation_config
    content: "<delegated administrator status>"
  - type: org_config
    content: "<Organizations and trusted service configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix trusted service access and register delegated admin"
  long_term: "Document delegation setup and implement monitoring"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never dismiss or archive health events without documenting their resolution
- Never disable EventBridge rules for health notifications in production accounts
- Never ignore scheduled maintenance events without planning for impact
- Never delete organizational health view configurations without stakeholder approval
- Never modify health event filters that may suppress critical notifications

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Re-enable EventBridge notification rules if health alerts stop flowing
- Restore previous event filter configurations if critical events are missed
- Revert organizational view delegation changes if visibility is lost

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
