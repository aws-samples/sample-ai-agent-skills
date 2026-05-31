---
title: "E2 — SCP and IAM Policy Interaction"
description: "How SCPs interact with IAM policies in the evaluation chain"
status: active
severity: MEDIUM
triggers:
  - "SCP and IAM"
  - "SCP evaluation order"
  - "SCP interaction"
  - "organization policy evaluation"
owner: devops-agent
objective: "Clarify how SCPs and IAM policies interact and diagnose issues at the intersection"
context: "SCPs set the maximum permissions for member accounts. They don't grant permissions — IAM policies do. For an action to succeed in a member account: the SCP must allow it AND the IAM policy must allow it AND no explicit deny exists in either. SCPs are evaluated before identity policies in the chain."
---

## Phase 1 — Triage

MUST:
- Identify if the account is in an Organization: `aws organizations describe-organization`
- List SCPs on the account and its OUs: `aws organizations list-policies-for-target --target-id <account-id> --filter SERVICE_CONTROL_POLICY`
- Check the IAM policy for the principal: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names <action>`
- Determine if the denial is from SCP or IAM by checking CloudTrail error details

SHOULD:
- Get the full SCP documents to check for both Allow and Deny statements
- Verify the FullAWSAccess SCP is attached (default allow-all SCP)
- Check if the SCP uses conditions that might not match the request context

MAY:
- Map the complete evaluation chain: SCP → resource policy → boundary → session → identity
- Test with a principal in the management account (SCP-exempt) to confirm SCP is the blocker

## Phase 2 — Remediate

MUST:
- If SCP is blocking: modify the SCP to allow the action (requires Organization admin)
- If IAM is blocking: add the permission to the identity policy (account-level)
- Both must allow — fixing only one side won't resolve the issue

SHOULD:
- Coordinate SCP changes with the Organization administrator
- Document the SCP-IAM interaction for the specific use case
- Test in a sandbox account before applying SCP changes broadly

MAY:
- Create an SCP exception using conditions (e.g., allow specific roles to bypass)
- Use AWS Control Tower guardrails instead of raw SCPs for common restrictions

## Common Issues

- symptoms: "IAM policy allows the action, simulate-principal-policy says allowed, but actual call is denied"
  diagnosis: "simulate-principal-policy does not evaluate SCPs. The SCP is blocking."
  resolution: "Check SCPs on the account and all parent OUs."

- symptoms: "SCP allows the action but IAM policy doesn't have it"
  diagnosis: "SCPs don't grant permissions. The IAM policy must also allow the action."
  resolution: "Add the permission to the IAM identity policy."

- symptoms: "Action works in one account but not another in the same OU"
  diagnosis: "The accounts may have different directly-attached SCPs, or different IAM policies."
  resolution: "Compare SCPs and IAM policies between the two accounts."

## Output Format

```yaml
root_cause: "scp_iam_interaction — <specific_cause>"
evidence:
  - type: scp_evaluation
    content: "<SCP allows/denies the action>"
  - type: iam_evaluation
    content: "<IAM policy allows/denies the action>"
  - type: combined_result
    content: "SCP: <allow|deny>, IAM: <allow|deny> → <final_result>"
severity: MEDIUM
mitigation:
  immediate: "Fix the blocking layer (SCP or IAM)"
  long_term: "Document SCP-IAM interaction patterns for the organization"
```


## Safety Ratings

safety_ratings:
  - "SCP-IAM interaction analysis (read-only evaluation of how SCPs and IAM policies interact): GREEN — All triage commands are read-only API calls that do not modify any resources"

## Escalation Conditions

escalation_conditions:
  - "Escalate to a human if the fix requires modifying an SCP (affects entire OU)"
  - "Escalate if the interaction involves multiple policy layers that cannot be resolved at the IAM level alone"
  - "Escalate if the issue affects multiple accounts in the organization"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Revert the IAM policy change by restoring the previous policy version, or revert the SCP change by restoring the original SCP document
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws organizations describe-policy` returns full SCP documents — classification: HIGH
- `aws iam simulate-principal-policy` reveals effective permissions — classification: HIGH
- `aws cloudtrail lookup-events` may reveal denied operations and caller details — classification: MEDIUM

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- No temporary resources are created during SCP-IAM evaluation analysis

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
