---
title: "Z1 — General IAM Troubleshooting"
description: "Catch-all systematic investigation for IAM issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "IAM issue"
  - "IAM problem"
  - "IAM not working"
  - "permission problem"
  - "something wrong with IAM"
owner: devops-agent
objective: "Systematically investigate any IAM issue using a structured approach"
context: "This is the catch-all runbook for IAM issues that don't clearly match a specific category. It provides a systematic investigation framework covering all IAM components: identity policies, resource policies, SCPs, permission boundaries, trust policies, credentials, and federation."
---

## Phase 1 — Triage

MUST:
- Identify the caller: `aws sts get-caller-identity`
- Identify the exact error message and API call that failed
- Determine the principal type (user, role, federated, service)
- Check CloudTrail for the event: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=<api-call> --max-results 5`
- Run policy simulation: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names <action> --resource-arns <resource>`

SHOULD:
- Collect all policies for the principal:
  - `aws iam list-attached-role-policies --role-name <role>` (or user equivalent)
  - `aws iam list-role-policies --role-name <role>` (inline)
  - `aws iam get-role --role-name <role>` (trust policy and boundary)
- Check for permission boundaries
- Check for SCPs if in an Organization
- Check resource-based policies on the target resource

MAY:
- Use IAM Access Analyzer: `aws accessanalyzer list-findings --analyzer-arn <arn>`
- Generate credential report: `aws iam generate-credential-report`
- Check account authorization details: `aws iam get-account-authorization-details`

## Phase 2 — Remediate

MUST:
- Follow the policy evaluation chain:
  1. Explicit Deny → check all policy types for Deny statements
  2. SCP → verify SCP allows the action
  3. Resource policy → check if it grants or denies
  4. Permission boundary → verify it allows the action
  5. Session policy → check if one was passed during AssumeRole
  6. Identity policy → verify it grants the action
- Fix the specific layer that is causing the issue
- Verify the fix with simulate-principal-policy and a test call

SHOULD:
- Document the root cause and fix for future reference
- Check if the fix introduces any unintended access
- Apply least-privilege principles

MAY:
- Set up monitoring for similar issues
- Implement preventive controls (SCPs, boundaries) to avoid recurrence

## Common Issues

- symptoms: "Everything looks correct but access is still denied"
  diagnosis: "Check: (1) IAM eventual consistency — wait 30-60 seconds, (2) VPC endpoint policy, (3) session policy passed during AssumeRole, (4) SCP at a parent OU level."
  resolution: "Systematically check each layer. The issue is usually in a layer that was overlooked."

- symptoms: "Access works intermittently"
  diagnosis: "IAM eventual consistency, DNS caching, or credential refresh timing."
  resolution: "Wait for propagation. Check if credentials are being refreshed properly."

- symptoms: "Cannot determine which policy is causing the deny"
  diagnosis: "Multiple overlapping policies make it hard to trace."
  resolution: "Use simulate-principal-policy with --context-entries for conditions. Check CloudTrail errorMessage for hints."

## Output Format

```yaml
root_cause: "<category> — <specific_cause>"
evidence:
  - type: caller_identity
    content: "<sts get-caller-identity>"
  - type: error_details
    content: "<error message and API call>"
  - type: policy_evaluation
    content: "<simulation result and policy chain analysis>"
severity: MEDIUM
mitigation:
  immediate: "<specific fix>"
  long_term: "<preventive measures>"
investigation_path:
  - step: "<what was checked>"
    result: "<finding>"
```


## Safety Ratings

safety_ratings:
  - "General IAM triage (systematic read-only investigation across all IAM components): GREEN — All triage commands are read-only API calls; this runbook focuses on diagnosis before directing to specific remediation runbooks"

## Escalation Conditions

escalation_conditions:
  - "Escalate to a human if the issue involves production IAM resources (roles, policies, or users serving live traffic)"
  - "Escalate if the root cause cannot be identified after systematic investigation of all policy layers"
  - "Escalate if the issue involves multiple interacting policy layers (SCP + boundary + identity policy)"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Revert the specific policy change by restoring the previous policy version using `aws iam create-policy-version --set-as-default` with the original document, or detach/delete the newly added policy
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws iam get-account-authorization-details` returns ALL policies, users, roles, and groups in the account — classification: HIGH
- `aws iam generate-credential-report` returns credential details for all IAM users — classification: HIGH
- `aws cloudtrail lookup-events` may return event details including IP addresses, user agents, and API parameters — classification: HIGH
- `aws iam simulate-principal-policy` reveals the effective permission model — classification: HIGH

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- Clean up any IAM Access Analyzer policy generation jobs started during investigation
- No other temporary resources are created during general troubleshooting

## Data Sensitivity

data_sensitivity:
  - command: "get-account-authorization-details"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-policy-version"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-role"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest Action: * or Resource: * in any policy"
  - "NEVER suggest removing permission boundaries"
  - "NEVER suggest AdministratorAccess for troubleshooting"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
