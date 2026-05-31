---
title: "E2 — Event Bus Policy Issues"
description: "Diagnose event bus resource policy configuration problems"
status: active
severity: MEDIUM
triggers:
  - "event bus policy"
  - "bus policy error"
  - "resource policy"
  - "policy denied"
  - "bus permissions"
owner: devops-agent
objective: "Identify and resolve event bus resource policy issues"
context: "Event bus resource policies control who can put events, manage rules, and access the bus. Policies use IAM policy syntax. Issues include overly restrictive policies, syntax errors, and missing cross-account permissions."
---

## Phase 1 — Triage

MUST:
- Describe the event bus: `aws events describe-event-bus --name <bus-name>`
- Review the resource policy (Policy field)
- Verify the policy syntax is valid JSON
- Check for specific deny statements that may block access

SHOULD:
- Verify principal ARNs in the policy are correct
- Check condition keys for unintended restrictions
- Review CloudTrail for policy modification history
- Test access from the intended principals

MAY:
- Use IAM Policy Simulator to evaluate the policy
- Check for organization-level policies
- Review policy size limits

## Phase 2 — Remediate

MUST:
- Fix policy syntax errors
- Add required permissions for cross-account or cross-service access
- Remove overly restrictive deny statements

SHOULD:
- Follow least-privilege principle in bus policies
- Use specific account IDs and conditions
- Document policy purpose and permissions

MAY:
- Implement policy versioning and change management
- Set up policy compliance monitoring

## Common Issues

- symptoms: "Cannot put events to custom event bus"
  diagnosis: "Event bus policy does not grant the caller events:PutEvents permission."
  resolution: "Add events:PutEvents permission for the caller's principal in the bus policy."

- symptoms: "Policy update fails with validation error"
  diagnosis: "Policy JSON syntax is invalid or exceeds size limits."
  resolution: "Validate JSON syntax. Check policy size (max 10240 characters). Simplify conditions."

- symptoms: "Partner events not arriving on the bus"
  diagnosis: "Event bus policy does not grant the partner event source permission."
  resolution: "Partner event sources require specific policy entries. Check partner documentation."

## Output Format

```yaml
root_cause: "event_bus_policy — <specific_cause>"
evidence:
  - type: bus_policy
    content: "<event bus resource policy>"
  - type: access_test
    content: "<access test results>"
  - type: cloudtrail
    content: "<policy modification history>"
severity: MEDIUM
mitigation:
  immediate: "Fix policy syntax or permissions"
  long_term: "Implement policy governance and documentation"
```


## Safety Ratings
```
safety_ratings:
  - "Describe event bus: GREEN — read-only API call"
  - "Review resource policy: GREEN — read-only analysis"
  - "Check CloudTrail for policy changes: GREEN — read-only audit log query"
  - "Add permissions to bus policy: YELLOW — expands access to the event bus"
  - "Remove deny statements: RED — may expose the bus to previously blocked access"
  - "Update policy with new conditions: YELLOW — changes access control logic"
```

## Escalation Conditions
- Event bus serves production event routing
- Policy changes affect cross-account or cross-service access
- Deny statements blocking legitimate access require security review
- Policy size limit reached requiring policy restructuring
- Partner event source access requires specific policy entries

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Resource policy: access control configuration with account IDs"
    - "Principal ARNs: identity information"
    - "Condition keys: access control logic"
  handling: "Event bus policies contain account IDs and access logic. Do not expose externally."
```

## Prohibited Actions
- NEVER use wildcard (*) principals in production event bus policies
- NEVER remove deny statements without understanding why they were added
- NEVER exceed the 10240 character policy size limit without restructuring
- NEVER modify the default event bus policy without understanding the blast radius

## Phase 3 — Rollback
- If policy was updated: restore previous policy using the saved policy JSON
- If deny statement was removed: re-add the deny statement
- If permissions were added: remove the added permission statements
- If policy was deleted: recreate the policy from backup or IaC source

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
