---
name: organizations-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Organizations problems
  by analyzing account management, SCPs, OU structure, delegated administration,
  billing, organization policies, service access, and following structured
  runbooks. Activate when: account creation failures, account closure issues,
  account suspension, SCP evaluation problems, SCP conflicts, SCP inheritance
  confusion, OU structure issues, OU limits, delegated administrator setup,
  trusted access problems, consolidated billing issues, cost allocation,
  tag policies, AI opt-out policies, backup policies, service integration
  failures, feature enablement, account migration, organization restructuring,
  or the user says something is wrong with AWS Organizations without naming
  specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Organizations, IAM, CloudTrail,
  CloudWatch, and optionally Control Tower, SSO, Service Catalog,
  and Cost Explorer permissions. Management account access required
  for most operations.
---

# AWS Organizations Diagnostics

## When to use

Any AWS Organizations investigation where the console alone is insufficient — account lifecycle management, SCP troubleshooting, OU structure debugging, delegated administration setup, billing analysis, organization policy management, service integration, or account migration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws organizations describe-organization
aws organizations list-accounts --query 'Accounts[*].{Id:Id,Name:Name,Status:Status,Email:Email}'
version: "1.0.0"
last_updated: "2025-04-12"
aws organizations list-roots
aws organizations list-organizational-units-for-parent --parent-id <root-id>
aws organizations list-policies --filter SERVICE_CONTROL_POLICY
aws organizations describe-account --account-id <account-id>
```

### Step 2 — Domain deep dive

```
aws organizations list-policies-for-target --target-id <target-id> --filter SERVICE_CONTROL_POLICY
aws organizations describe-policy --policy-id <policy-id>
aws organizations list-targets-for-policy --policy-id <policy-id>
aws organizations list-children --parent-id <ou-id> --child-type ACCOUNT
aws organizations list-delegated-administrators
aws organizations list-aws-service-access-for-organization
```

### Step 3 — Detailed investigation

```
aws organizations describe-effective-policy --policy-type TAG_POLICY --target-id <account-id>
aws organizations list-delegated-services-for-account --account-id <account-id>
aws organizations describe-create-account-status --create-account-request-id <request-id>
aws organizations list-handshakes-for-organization
aws organizations list-create-account-status --states FAILED
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=organizations.amazonaws.com --max-results 20
```

Read `references/organizations-guardrails.md` before concluding on any Organizations issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-organization` | Organization details, feature set, master account |
| `list-accounts` | All accounts in the organization |
| `describe-account` | Specific account details and status |
| `list-roots` | Organization root(s) |
| `list-organizational-units-for-parent` | OUs under a parent |
| `list-policies` | Policies by type (SCP, TAG_POLICY, etc.) |
| `describe-policy` | Policy content and metadata |
| `list-policies-for-target` | Policies attached to a target |
| `list-targets-for-policy` | Targets a policy is attached to |
| `describe-effective-policy` | Effective merged policy for a target |
| `list-delegated-administrators` | Delegated admin accounts |
| `list-aws-service-access-for-organization` | Enabled service integrations |
| `list-create-account-status` | Account creation request status |
| `list-handshakes-for-organization` | Pending invitations and handshakes |
| `describe-create-account-status` | Specific account creation status |

## Gotchas: AWS Organizations

- Management account is exempt from SCPs. The management (formerly master) account is NEVER affected by SCPs, regardless of what SCPs are attached to the root or any OU. This is by design to prevent lockout. Only member accounts are subject to SCPs. Never test SCP behavior from the management account.
- SCP is a deny filter, not a grant. SCPs do not grant permissions — they define the maximum permissions boundary. An SCP Allow statement does not grant access; it only permits the action to be allowed by IAM policies. An SCP Deny statement blocks the action regardless of IAM policies. The effective permissions are the intersection of SCP allows and IAM grants.
- Account email is globally unique. Each AWS account email must be globally unique across ALL AWS accounts worldwide. You cannot reuse an email from a closed account until the closure is fully processed. Use email aliases (user+tag@domain.com) for creating multiple accounts with one mailbox.
- Account closure has a 90-day recovery period. After closing an account, there is a 90-day grace period during which the account can be reopened. After 90 days, the account is permanently closed and all resources are deleted. During the grace period, the account is suspended and cannot be used.
- Delegated administrator is limited to specific services. Not all AWS services support delegated administration. Each service has its own delegation model. A maximum of 5 delegated administrators per service is typical. The management account must enable delegation for each service.
- OU nesting maximum is 5 levels. OUs can be nested up to 5 levels deep (root + 5 OU levels). The root counts as the first level. Exceeding this limit prevents creating deeper OUs. Plan the OU hierarchy carefully as restructuring requires moving accounts.
- SCP maximum is 5 per target. Each target (root, OU, or account) can have a maximum of 5 SCPs attached directly. Inherited SCPs from parent OUs do not count against this limit. The FullAWSAccess default SCP counts as one of the 5.
- Policy inheritance is cumulative — all ancestors apply. SCPs are inherited from all parent OUs up to the root. The effective SCP is the intersection of all inherited SCPs. A deny at any level blocks the action for all descendants. Moving an account between OUs changes its effective SCPs immediately.
- Consolidated billing shares RI/SP benefits. Reserved Instances and Savings Plans purchased in any account are shared across the organization by default. This can be disabled per account. Cost allocation tags must be activated at the management account level.
- Removing an account from the org requires root email access. To remove a member account (make it standalone), the account must have a valid payment method, root email access, and support plan. Accounts created via Organizations may not have root credentials set up — you must go through the password reset flow using the account email.

## Anti-hallucination rules

1. Always cite specific account IDs, policy IDs, or API responses as evidence.
2. Management account is exempt from SCPs. Never suggest testing SCPs from the management account.
3. SCPs are deny filters, not grants. Never suggest an SCP Allow statement grants permissions.
4. Account closure has a 90-day recovery period. Never suggest immediate permanent deletion.
5. OU nesting is limited to 5 levels. Never suggest deeper hierarchies.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 26 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Accounts | A1-A3 | Account creation failures, account closure, account suspension |
| B — SCPs | B1-B3 | SCP evaluation, SCP conflicts, SCP inheritance |
| C — OUs | C1-C2 | OU structure issues, OU limits |
| D — Delegation | D1-D2 | Delegated administrator, trusted access |
| E — Billing | E1-E2 | Consolidated billing, cost allocation |
| F — Policies | F1-F3 | Tag policies, AI opt-out, backup policies |
| G — Service Access | G1-G2 | Service integration, feature enablement |
| H — Migration | H1-H2 | Account migration, organization restructuring |
| Z — Catch-All | Z1 | General troubleshooting |
