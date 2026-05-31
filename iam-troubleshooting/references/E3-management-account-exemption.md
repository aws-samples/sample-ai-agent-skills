---
title: "E3 — Management Account SCP Exemption"
description: "Understanding that SCPs do not apply to the management account"
status: active
severity: MEDIUM
triggers:
  - "management account"
  - "master account"
  - "SCP exemption"
  - "SCP not applying"
  - "works in management account"
owner: devops-agent
objective: "Clarify management account SCP exemption and diagnose related confusion"
context: "The management (formerly master) account in AWS Organizations is exempt from all SCPs. Actions that are denied by SCPs in member accounts will succeed in the management account. This is by design and cannot be changed. This often causes confusion when testing policies."
---

## Phase 1 — Triage

MUST:
- Identify if the account is the management account: `aws organizations describe-organization` (look for MasterAccountId)
- Confirm the behavior difference: action works in management account but fails in member account
- Verify SCPs are the cause by checking SCP policies on the member account

SHOULD:
- Check if the user is testing in the management account and expecting the same behavior in member accounts
- Verify the member account has the correct SCPs attached
- Document which actions are SCP-restricted

MAY:
- Review the organization's SCP strategy for management account governance
- Check if AWS Control Tower is managing the organization

## Phase 2 — Remediate

MUST:
- Accept that SCPs cannot restrict the management account — this is an AWS design decision
- For member accounts: ensure SCPs allow the required actions
- Do not use the management account for workloads — use it only for organization management

SHOULD:
- Test SCP effects in member accounts, not the management account
- Implement IAM policies in the management account as a substitute for SCP restrictions
- Minimize the number of users/roles in the management account

MAY:
- Use AWS Control Tower to enforce governance on the management account through other mechanisms
- Implement CloudWatch alarms for sensitive actions in the management account
- Use AWS Config rules to detect non-compliant resources in the management account

## Common Issues

- symptoms: "SCP denies ec2:RunInstances but it works in one account"
  diagnosis: "The working account is the management account, which is SCP-exempt."
  resolution: "This is expected. Use IAM policies in the management account for restrictions."

- symptoms: "Created SCP to deny all but it doesn't affect the management account"
  diagnosis: "SCPs never apply to the management account."
  resolution: "Use IAM policies with explicit Deny in the management account instead."

- symptoms: "Service-linked roles bypass SCPs in member accounts"
  diagnosis: "SCPs don't apply to service-linked roles. This is separate from the management account exemption."
  resolution: "This is expected. SLRs are managed by AWS services and need unrestricted access."

## Output Format

```yaml
root_cause: "management_account_exemption — <specific_scenario>"
evidence:
  - type: account_type
    content: "management_account | member_account"
  - type: scp_status
    content: "<SCP applies: yes/no>"
  - type: behavior_difference
    content: "<action succeeds in mgmt, fails in member>"
severity: MEDIUM
mitigation:
  immediate: "Use IAM policies for management account restrictions"
  long_term: "Minimize management account usage and implement compensating controls"
```


## Safety Ratings

safety_ratings:
  - "Management account SCP exemption analysis (informational guidance on SCP behavior): GREEN — This runbook provides informational guidance and does not modify any resources"

## Escalation Conditions

escalation_conditions:
  - "Escalate to a human if the user is running workloads in the management account (this is an anti-pattern)"
  - "Escalate if the user needs to restrict actions in the management account (requires IAM policies instead of SCPs)"
  - "Escalate if the user is confused about SCP behavior and may make incorrect assumptions about security controls"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Revert any IAM policy changes made in the management account by restoring the previous policy version or detaching the added policy
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws organizations describe-organization` reveals the management account ID and organization structure — classification: MEDIUM
- Management account IAM policies may contain sensitive organizational governance rules — classification: HIGH

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- No temporary resources are created during management account exemption analysis

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
