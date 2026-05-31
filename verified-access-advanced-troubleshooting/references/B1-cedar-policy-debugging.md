---
title: "B1 — Cedar Policy Debugging"
description: "Debug Cedar policy syntax and logic errors in Verified Access"
status: active
severity: HIGH
triggers: ["Cedar policy error", "policy syntax error", "permit not working", "forbid not matching", "policy debugging"]
owner: devops-agent
objective: "Resolve Cedar policy syntax errors and logic issues in Verified Access groups"
context: "Cedar policies control access decisions in Verified Access. They use permit/forbid statements with when/unless clauses referencing trust provider context data."
---

## Phase 1 — Triage

MUST:
- Get current policy: `aws ec2 get-verified-access-group-policy --verified-access-group-id <group-id>`
- Check access logs for policy evaluation details: `aws logs filter-log-events --log-group-name <va-log-group> --filter-pattern '{ $.activity = "denied" }' --start-time <epoch-ms>`
- Verify trust provider policy_reference_names: `aws ec2 describe-verified-access-trust-providers --query 'VerifiedAccessTrustProviders[*].{Id:VerifiedAccessTrustProviderId,PolicyRef:PolicyReferenceName}'`

SHOULD:
- Review Cedar syntax for common errors (missing semicolons, incorrect operators)
- Verify context path references match actual trust provider data structure
- Check if multiple policies conflict (permit vs forbid precedence)

MAY:
- Test with a minimal permit-all policy to isolate policy vs connectivity issues
- Review Cedar language documentation for operator usage

## Phase 2 — Remediate

MUST:
- Fix Cedar syntax errors (ensure statements end with semicolons)
- Use correct context path: `context.<policy_reference_name>.<claim_name>`
- Ensure at least one permit statement matches the intended access pattern

SHOULD:
- Test policies with known-good user credentials before deploying broadly
- Use access logs to verify policy evaluation results after changes

MAY:
- Implement policy versioning via IaC for rollback capability
- Create policy templates for common access patterns

## Common Issues

- symptoms: "All access denied despite permit policy"
  diagnosis: "Context path in Cedar policy does not match trust provider data structure."
  resolution: "Check access logs for actual context data. Verify context.<policy_reference_name> matches the trust provider's PolicyReferenceName."

## Safety Ratings

```
safety_ratings:
  - "get-verified-access-group-policy: GREEN — read-only policy retrieval"
  - "modify-verified-access-group-policy: YELLOW — changes authorization rules"
  - "Access log analysis: GREEN — read-only log review"
  - "Permit-all test policy: RED — temporarily removes access controls"
```

## Escalation Conditions

- Cedar policy changes cause widespread access denials in production
- Policy logic is too complex to debug without Cedar language expertise
- Trust provider context data structure is undocumented or unexpected
- Policy evaluation results are inconsistent or non-deterministic

## Data Sensitivity

- `aws ec2 get-verified-access-group-policy`: MEDIUM — reveals authorization logic
- Access logs: HIGH — contain user identity, device posture, and access decisions
- Cedar policy content: MEDIUM — reveals access control rules
- Trust context data: HIGH — contains identity claims and device information

## Prohibited Actions

- NEVER deploy a permit-all policy in production without a time-limited rollback plan
- NEVER modify Cedar policies without checking access logs first
- NEVER use IAM policy JSON syntax in Cedar policies
- NEVER remove forbid statements without understanding their security purpose

## Phase 3 — Rollback

1. Revert to previous Cedar policy: `aws ec2 modify-verified-access-group-policy --verified-access-group-id <group-id> --policy-document "<previous-policy>"`
2. If the previous policy is unknown, check CloudTrail for the last ModifyVerifiedAccessGroupPolicy event
3. Verify access is restored for legitimate users after rollback
4. Check access logs to confirm policy evaluation matches expectations
5. Document the policy change and rollback for audit trail

## Output Format

```yaml
root_cause: "cedar_policy — <specific_cause>"
evidence:
  - type: policy_evaluation
    content: "<Cedar policy and access log details>"
severity: HIGH
mitigation:
  immediate: "Fix Cedar policy syntax or logic"
  long_term: "Implement policy testing and versioning"
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
