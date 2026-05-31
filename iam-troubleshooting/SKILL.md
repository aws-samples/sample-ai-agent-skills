---
name: iam-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS IAM problems
  by analyzing policies, roles, trust relationships, and following
  structured runbooks. Activate when: access denied errors, policy
  evaluation confusion, role assumption failures, cross-account
  access issues, permission boundary conflicts, SCP restrictions,
  MFA enforcement problems, credential rotation issues, federation
  failures, service-linked role issues, or the user says something
  is wrong with IAM without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with IAM, STS, Organizations,
  CloudTrail, and IAM Access Analyzer permissions.
---

# IAM Diagnostics

## When to use

Any IAM investigation where the console alone is insufficient — access denied root cause analysis, policy evaluation logic, role trust chain debugging, cross-account access, permission boundaries, SCP interactions, federation issues, or credential problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws iam get-user --user-name <user>
aws iam list-attached-user-policies --user-name <user>
aws iam list-user-policies --user-name <user>
aws iam list-groups-for-user --user-name <user>
aws sts get-caller-identity
aws iam simulate-principal-policy --policy-source-arn <arn> --action-names <action> --resource-arns <resource>
```

### Step 2 — Domain deep dive

```
aws iam get-role --role-name <role>
aws iam get-policy-version --policy-arn <arn> --version-id <version>
aws iam list-role-policies --role-name <role>
aws iam get-account-authorization-details
aws accessanalyzer list-findings --analyzer-arn <arn>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole
```

Read `references/iam-guardrails.md` before concluding on any IAM issue.

## Gotchas: IAM

- Policy evaluation order: explicit Deny → SCP → resource policy → permission boundary → session policy → identity policy. Explicit deny in ANY layer wins.
- IAM policies have a 6,144 character limit (managed) or 10,240 (inline). Complex policies hit this.
- Permission boundaries set the MAXIMUM permissions. They don't grant permissions — identity policies do that. Effective permissions = intersection of boundary and identity policy.
- SCPs don't apply to the management account or service-linked roles.
- Role trust policies control WHO can assume the role. The assuming principal also needs sts:AssumeRole permission.
- Session policies (passed during AssumeRole) further restrict the role's permissions for that session. They don't grant additional permissions.
- IAM changes are eventually consistent. A policy change may take seconds to propagate globally.
- Wildcard (*) in Principal means EVERYONE including anonymous. Use specific account/role ARNs.
- Condition keys are case-sensitive. aws:SourceIp ≠ AWS:sourceip.
- Cross-account role assumption: the trust policy must allow the external account, AND the external account must have sts:AssumeRole permission.
- Service-linked roles are managed by AWS services. You cannot modify their permissions. You can only delete them if the service allows it.
- Access keys should be rotated regularly. Old keys remain active until explicitly deactivated.
- MFA conditions (aws:MultiFactorAuthPresent) don't work with long-term access keys in CLI — only with temporary credentials from STS.

### Policy evaluation logic

| Step | Check | Effect |
|------|-------|--------|
| 1 | Explicit Deny in any policy | DENY (final) |
| 2 | SCP (if in Organization) | Must ALLOW (or implicit deny) |
| 3 | Resource-based policy | Can ALLOW (even without identity policy for same-account) |
| 4 | Permission boundary | Must ALLOW (if set) |
| 5 | Session policy | Must ALLOW (if set) |
| 6 | Identity-based policy | Must ALLOW |

## Anti-hallucination rules

1. Always cite specific policy documents, API responses, or CloudTrail events as evidence.
2. Permission boundaries restrict, they don't grant. Never claim a boundary grants permissions.
3. SCPs don't apply to the management account. Never claim SCPs restrict the management account.
4. Policy evaluation has a specific order. Never skip layers in the evaluation.
5. IAM changes are eventually consistent. Never claim changes are instant.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 30 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Access Denied | A1-A4 | Policy evaluation, explicit deny, implicit deny, resource policy |
| B — Role Issues | B1-B4 | Trust policy, cross-account assume, session duration, chained roles |
| C — Policy Management | C1-C3 | Policy size limits, policy versioning, managed vs inline |
| D — Permission Boundaries | D1-D2 | Boundary conflicts, effective permissions |
| E — SCP Interactions | E1-E3 | SCP deny, SCP + IAM evaluation, management account exemption |
| F — Federation | F1-F3 | SAML federation, OIDC federation, web identity |
| G — Credentials | G1-G3 | Access key rotation, temporary credentials, MFA issues |
| H — Cross-Account | H1-H3 | Cross-account roles, resource sharing, confused deputy |
| I — Service Roles | I1-I2 | Service-linked roles, service roles |
| Z — Catch-All | Z1 | General troubleshooting |
