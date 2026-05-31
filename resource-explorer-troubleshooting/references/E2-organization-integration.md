---
title: "E2 — Organization Integration"
description: "Diagnose AWS Organizations integration issues with Resource Explorer"
status: active
severity: HIGH
triggers: ["organization integration", "multi-account search", "delegated admin", "org-wide resource search"]
owner: devops-agent
objective: "Resolve organization-level Resource Explorer integration issues"
context: "Multi-account resource search requires Organizations integration with delegated administrator setup."
---

## Phase 1 — Triage

MUST:
- Verify Organizations is enabled: `aws organizations describe-organization`
- Check delegated administrator status: `aws organizations list-delegated-administrators --service-principal resource-explorer-2.amazonaws.com`
- Verify multi-account search is configured: `aws resource-explorer-2 get-account-level-service-configuration`

SHOULD:
- Check if the caller is in the management account or delegated admin account
- Verify member accounts have indexes created

MAY:
- Review CloudTrail for organization-related Resource Explorer events
- Check SCPs that may block Resource Explorer in member accounts

## Phase 2 — Remediate

MUST:
- Register delegated administrator if not configured: `aws organizations register-delegated-administrator --account-id <id> --service-principal resource-explorer-2.amazonaws.com`
- Enable multi-account search from the management or delegated admin account
- Ensure member accounts have local indexes

SHOULD:
- Use StackSets to deploy indexes across member accounts
- Configure the aggregator in the delegated admin account

MAY:
- Set up automated onboarding for new member accounts
- Create organization-wide views for cross-account search

## Safety Ratings

```
safety_ratings:
  - "organizations describe-organization: GREEN — read-only org details"
  - "organizations list-delegated-administrators: GREEN — read-only admin list"
  - "organizations register-delegated-administrator: RED — grants org-level admin privileges"
  - "resource-explorer-2 get-account-level-service-configuration: GREEN — read-only config"
```

## Escalation Conditions

- Delegated administrator registration requires management account access
- SCPs block Resource Explorer in member accounts
- Multi-account search returns incomplete results across the organization
- Organization structure changes affect Resource Explorer integration

## Data Sensitivity

- `aws organizations describe-organization`: MEDIUM — reveals organization structure
- `aws organizations list-delegated-administrators`: MEDIUM — reveals admin account IDs
- Multi-account search results: HIGH — reveals resource inventory across all accounts

## Prohibited Actions

- NEVER register a delegated administrator without management account owner approval
- NEVER enable multi-account search without understanding data visibility implications
- NEVER modify SCPs to fix Resource Explorer without security team review
- NEVER grant organization-level permissions to troubleshoot member account issues

## Phase 3 — Rollback

1. If delegated admin was registered incorrectly: `aws organizations deregister-delegated-administrator --account-id <id> --service-principal resource-explorer-2.amazonaws.com`
2. If multi-account search was enabled incorrectly, disable it from the admin account
3. Verify member account indexes are unaffected by organization changes
4. Restore SCPs if they were modified during troubleshooting
5. Notify the security team of any organization-level changes

## Common Issues

- symptoms: "Cannot enable multi-account search — 'Not a delegated administrator'"
  diagnosis: "The caller account is not registered as a delegated administrator for Resource Explorer."
  resolution: "Register from management account: aws organizations register-delegated-administrator --account-id <admin-account> --service-principal resource-explorer-2.amazonaws.com"

## Output Format

```yaml
root_cause: "organization_integration — <specific_cause>"
evidence:
  - type: org_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix organization integration"
  long_term: "Automate member account onboarding"
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
