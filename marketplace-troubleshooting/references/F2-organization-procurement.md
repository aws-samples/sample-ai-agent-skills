---
title: "F2 — Organization Procurement"
description: "Diagnose organization-level procurement and governance issues"
status: active
severity: MEDIUM
triggers: ["org procurement", "organization marketplace", "multi-account procurement", "procurement governance"]
owner: devops-agent
objective: "Resolve organization-level procurement issues"
context: "Organization procurement involves managing Marketplace subscriptions across multiple accounts using AWS Organizations, SCPs, and Private Marketplace."
---

## Phase 1 — Triage

MUST:
- Check Organizations service access: `aws organizations list-aws-service-access-for-organization`
- Verify SCPs affecting marketplace actions: `aws organizations list-policies --filter SERVICE_CONTROL_POLICY`
- Check delegated admin for Marketplace: `aws organizations list-delegated-administrators --service-principal aws-marketplace.amazonaws.com`

SHOULD:
- Review SCP content for marketplace restrictions: `aws organizations describe-policy --policy-id <policy-id>`
- Check OU structure for procurement governance: `aws organizations list-organizational-units-for-parent --parent-id <root-id>`

MAY:
- Verify consolidated billing setup
- Check for tag policies affecting Marketplace resources

## Phase 2 — Remediate

MUST:
- Enable Organizations service access for Marketplace: `aws organizations enable-aws-service-access --service-principal aws-marketplace.amazonaws.com`
- Fix SCPs blocking legitimate procurement
- Set up delegated admin if management account should not handle procurement

SHOULD:
- Register delegated admin: `aws organizations register-delegated-administrator --account-id <id> --service-principal aws-marketplace.amazonaws.com`
- Configure SCPs to allow approved procurement patterns

MAY:
- Set up tag policies for Marketplace resource tagging
- Implement budget controls per OU for Marketplace spending

## Safety Ratings

```
safety_ratings:
  - "organizations list-aws-service-access: GREEN — read-only check"
  - "organizations list-policies: GREEN — read-only listing"
  - "organizations enable-aws-service-access: YELLOW — enables org integration"
  - "organizations register-delegated-administrator: RED — grants admin privileges"
```

## Escalation Conditions

- SCP changes affect procurement across all member accounts
- Delegated admin registration requires management account access
- Procurement governance conflicts between OUs
- Budget controls block critical software procurement

## Data Sensitivity

- `aws organizations list-policies`: LOW — returns policy metadata
- SCP content: MEDIUM — reveals security governance rules
- Organization structure: MEDIUM — reveals account hierarchy
- Delegated admin configuration: HIGH — affects org-level trust

## Prohibited Actions

- NEVER modify SCPs without understanding impact on all member accounts
- NEVER register delegated admin without management account owner approval
- NEVER disable Organizations service access without coordinating with all teams
- NEVER bypass procurement governance for convenience

## Phase 3 — Rollback

1. Revert SCP changes: `aws organizations update-policy --policy-id <id> --content file://original-policy.json`
2. Deregister delegated admin if registered in error: `aws organizations deregister-delegated-administrator --account-id <id> --service-principal aws-marketplace.amazonaws.com`
3. Disable service access if enabled incorrectly: `aws organizations disable-aws-service-access --service-principal aws-marketplace.amazonaws.com`
4. Verify member account procurement capabilities after changes
5. Document governance changes and notify affected teams

## Common Issues

- symptoms: "Member accounts get AccessDeniedException when subscribing"
  diagnosis: "SCP blocking aws-marketplace:Subscribe action."
  resolution: "Review and update SCP to allow marketplace actions for approved accounts/OUs."

## Output Format

```yaml
root_cause: "org_procurement — <specific_cause>"
evidence:
  - type: governance_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix organization procurement governance"
  long_term: "Implement structured procurement policies per OU"
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
