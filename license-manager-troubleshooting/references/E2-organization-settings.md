---
title: "E2 — Organization Settings"
description: "Diagnose License Manager organization-level settings issues"
status: active
severity: MEDIUM
triggers: ["org settings", "organization license manager", "delegated admin license", "org-wide licensing"]
owner: devops-agent
objective: "Resolve License Manager organization settings issues"
context: "Organization settings control cross-account discovery, sharing, and delegated admin. Must be configured from the management account."
---

## Phase 1 — Triage

MUST:
- Check service settings: `aws license-manager get-service-settings`
- Verify Organizations integration: `aws organizations list-aws-service-access-for-organization --query 'EnabledServicePrincipals[?ServicePrincipal==`license-manager.amazonaws.com`]'`
- Check delegated admin: `aws organizations list-delegated-administrators --service-principal license-manager.amazonaws.com`

SHOULD:
- Verify caller is management account or delegated admin: `aws sts get-caller-identity`
- Check for SCPs affecting License Manager

MAY:
- Review organization structure for OU-level settings
- Check if License Manager is enabled in all required regions

## Phase 2 — Remediate

MUST:
- Enable Organizations service access: `aws organizations enable-aws-service-access --service-principal license-manager.amazonaws.com`
- Configure service settings from management account: `aws license-manager update-service-settings --enable-cross-account-discovery`
- Set up delegated admin if needed: `aws organizations register-delegated-administrator --account-id <id> --service-principal license-manager.amazonaws.com`

SHOULD:
- Enable cross-account discovery for license tracking
- Configure S3 bucket for license manager reports

MAY:
- Set up automated compliance reporting
- Configure SNS notifications for organization-wide license events

## Safety Ratings

```
safety_ratings:
  - "license-manager get-service-settings: GREEN — read-only check"
  - "organizations list-aws-service-access: GREEN — read-only listing"
  - "organizations enable-aws-service-access: YELLOW — enables org integration"
  - "organizations register-delegated-administrator: RED — grants admin privileges"
```

## Escalation Conditions

- Management account access required but not available
- Organization settings changes affect all member accounts
- Delegated admin registration conflicts with other services
- Cross-account discovery reveals compliance issues

## Data Sensitivity

- Service settings: LOW — contains configuration metadata
- Organization structure: MEDIUM — reveals account hierarchy
- Delegated admin configuration: HIGH — affects org-level trust
- Cross-account discovery data: MEDIUM — reveals org-wide licensing

## Prohibited Actions

- NEVER modify organization settings without management account owner approval
- NEVER register delegated admin without understanding the scope of access
- NEVER disable cross-account discovery during compliance audits
- NEVER change organization settings that affect other AWS services

## Phase 3 — Rollback

1. Deregister delegated admin if registered in error: `aws organizations deregister-delegated-administrator --account-id <id> --service-principal license-manager.amazonaws.com`
2. Disable service access if enabled incorrectly: `aws organizations disable-aws-service-access --service-principal license-manager.amazonaws.com`
3. Revert service settings to previous configuration
4. Verify member accounts are not affected by changes
5. Document organization settings changes

## Common Issues

- symptoms: "UpdateServiceSettings returns AccessDeniedException"
  diagnosis: "Caller is not the management account or delegated admin."
  resolution: "Use management account credentials or register a delegated admin first."

## Output Format

```yaml
root_cause: "organization_settings — <specific_cause>"
evidence:
  - type: settings_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix organization settings"
  long_term: "Document and automate org settings management"
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
