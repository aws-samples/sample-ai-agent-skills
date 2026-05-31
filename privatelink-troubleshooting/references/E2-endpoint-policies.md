---
title: "E2 — Endpoint Policies"
description: "Diagnose VPC endpoint policy configuration and access control issues"
status: active
severity: MEDIUM
triggers:
  - "endpoint policy"
  - "policy blocking"
  - "access denied through endpoint"
  - "endpoint policy document"
  - "restrict endpoint access"
owner: devops-agent
objective: "Resolve endpoint policy issues that block or incorrectly restrict access"
context: "Endpoint policies are resource policies attached to VPC endpoints. They control which principals can use the endpoint to access the service. The default policy allows full access. Endpoint policies work IN ADDITION to IAM policies — both must allow the action. Endpoint policies use standard IAM policy syntax but are evaluated separately."
---

## Phase 1 — Triage

MUST:
- Check current endpoint policy: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id> --query 'VpcEndpoints[*].PolicyDocument'`
- Verify the policy allows the required actions and principals
- Check IAM policy for the caller (both must allow)
- Identify the specific action being denied

SHOULD:
- Compare the endpoint policy with the IAM policy to find the restriction
- Check CloudTrail for access denied events with VPC endpoint context
- Verify the policy JSON syntax is valid

MAY:
- Test with the default (full access) policy to confirm endpoint policy is the issue
- Review policy conditions for IP, VPC, or time restrictions

## Phase 2 — Remediate

MUST:
- Update endpoint policy to allow required actions: `aws ec2 modify-vpc-endpoint --vpc-endpoint-id <endpoint-id> --policy-document file://policy.json`
- Ensure both endpoint policy AND IAM policy allow the action
- Use the correct principal ARN format in the policy

SHOULD:
- Follow least-privilege principle in endpoint policies
- Test policy changes before applying to production endpoints
- Document endpoint policy requirements for each service

MAY:
- Implement policy management via IaC
- Set up Config rules to detect endpoint policy drift

## Common Issues

- symptoms: "Access denied only when using the VPC endpoint"
  diagnosis: "Endpoint policy is more restrictive than IAM policy."
  resolution: "Update endpoint policy to allow the required actions and principals."

- symptoms: "Default policy works but custom policy blocks access"
  diagnosis: "Custom policy syntax error or missing required actions."
  resolution: "Validate policy JSON. Ensure all required actions are listed. Check principal format."

- symptoms: "Endpoint policy allows * but access still denied"
  diagnosis: "IAM policy is denying the action. Endpoint policy cannot override IAM deny."
  resolution: "Check IAM policy. Endpoint policy allows but IAM must also allow."

## Output Format

```yaml
root_cause: "endpoint_policy — <specific_cause>"
evidence:
  - type: endpoint_policy
    content: "<current endpoint policy document>"
  - type: iam_policy
    content: "<caller IAM policy>"
  - type: denied_action
    content: "<specific action being denied>"
severity: MEDIUM
mitigation:
  immediate: "Update endpoint policy to allow required access"
  long_term: "Implement policy management and drift detection"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — endpoint policies control data access
- Service Disruption Risk: MEDIUM — restrictive policies block access
- Compliance Impact: HIGH — policies enforce data access compliance

## Escalation Conditions
- Policy changes block critical service access
- Overly permissive policies detected
- Policy syntax errors cause evaluation failures
- Compliance audit flags policy violations
- Policy changes affect multiple endpoints

## Data Sensitivity
- Policy documents — CONFIDENTIAL
- Access control configurations — CONFIDENTIAL
- Policy evaluation results — INTERNAL
- Service access patterns — INTERNAL
- Never expose policy documents externally

## Prohibited Actions
- Do NOT apply overly permissive endpoint policies
- Do NOT remove policies without understanding impact
- Do NOT bypass policy review processes
- Do NOT create policies allowing unauthorized access
- Do NOT ignore policy evaluation errors
- Do NOT modify policies without testing

## Phase 3 — Rollback
- Restore previous endpoint policies
- Revert policy changes that blocked access
- Verify access after policy changes
- Restore compliance-required controls
- Document policy changes and impact
- Update policy management procedures

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
