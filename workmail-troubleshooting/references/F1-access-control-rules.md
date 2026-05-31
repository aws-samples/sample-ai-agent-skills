---
title: "F1 — Access Control Rules"
description: "Diagnose WorkMail access control rule issues"
status: active
severity: HIGH
triggers: ["access blocked", "cannot login", "IP blocked", "protocol restricted", "access control error"]
owner: devops-agent
objective: "Resolve access control rule issues including IP restrictions, protocol blocking, and rule ordering"
context: "Access control rules restrict who can access WorkMail based on IP, protocol, and user. Rules are evaluated in order — first match wins."
---

## Phase 1 — Triage

MUST:
- List access control rules: `aws workmail list-access-control-rules --organization-id <org-id>`
- Identify which rule is blocking the user (check IP, protocol, and user criteria)
- Verify the user's IP address and client protocol
- Check rule evaluation order (first matching rule determines access)

SHOULD:
- Test access from a known-allowed IP to confirm it's a rule issue
- Check if the default rule (last rule) allows or denies access
- Verify the user is not in a blocked user list

MAY:
- Review CloudTrail for access control rule changes
- Check if recent rule changes correlate with the access issue

## Phase 2 — Remediate

MUST:
- Modify or reorder rules to allow legitimate access
- Add allow rules before deny rules for specific users or IPs
- Ensure the default rule provides appropriate fallback behavior

SHOULD:
- Test rule changes with a non-production user first
- Document the intended access control policy
- Verify all legitimate access patterns are covered by allow rules

MAY:
- Create separate rules for different user groups
- Set up monitoring for access control denials

## Safety Ratings

```
safety_ratings:
  - "workmail list-access-control-rules: GREEN — read-only rule listing"
  - "workmail put-access-control-rule: RED — modifies access control, can lock out users"
  - "workmail delete-access-control-rule: RED — removes access restriction, may open access"
  - "workmail get-access-control-effect: GREEN — tests rule evaluation without changes"
```

## Escalation Conditions

- Access control rules lock out all users including admins
- Rules cannot be modified due to API errors
- Complex rule interactions cause unpredictable access behavior
- Compliance requirements conflict with user access needs

## Data Sensitivity

- `aws workmail list-access-control-rules`: MEDIUM — reveals access control policies
- `aws workmail get-access-control-effect`: LOW — tests rule evaluation
- User IP addresses: MEDIUM — reveals user network locations
- Access patterns: MEDIUM — reveals organizational access behavior

## Prohibited Actions

- NEVER delete all access control rules without understanding the default behavior
- NEVER create deny-all rules without an admin exception
- NEVER modify access control rules during business hours without a rollback plan
- NEVER test rule changes on production users without a backup access method

## Phase 3 — Rollback

1. If a rule change locked out users, delete the problematic rule: `aws workmail delete-access-control-rule --organization-id <org-id> --name <rule-name>`
2. If rules were reordered incorrectly, restore original order by recreating rules
3. If the default rule was changed, restore it to the original allow/deny setting
4. Verify all users can access WorkMail after rule changes
5. Document the access control rule configuration for disaster recovery

## Common Issues

- symptoms: "User cannot access WorkMail from home network"
  diagnosis: "Access control rule restricts access to office IP ranges only."
  resolution: "Add the user's home IP range to an allow rule, or create a user-specific exception rule."

- symptoms: "Mobile devices blocked but web client works"
  diagnosis: "Access control rule blocks ActiveSync protocol."
  resolution: "Add an allow rule for ActiveSync protocol before the deny rule. Check rule order."

## Output Format

```yaml
root_cause: "access_control — <specific_cause>"
evidence:
  - type: rule_issue
    content: "<rule configuration, user IP, protocol, evaluation result>"
severity: HIGH
mitigation:
  immediate: "Fix access control rules"
  long_term: "Document and regularly review access control policies"
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
