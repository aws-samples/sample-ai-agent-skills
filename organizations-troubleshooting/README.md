# AWS Organizations Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Organizations problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Organizations when the console alone isn't enough — account lifecycle management, SCP evaluation and debugging, OU structure issues, delegated administration setup, consolidated billing problems, organization policy management, service integration, and account migration.

### Activate When

- Account creation failures
- Account closure issues
- Account suspension or reactivation
- SCP evaluation confusion
- SCP conflicts between policies
- SCP inheritance issues
- OU structure problems
- OU nesting or count limits
- Delegated administrator setup failures
- Trusted access configuration issues
- Consolidated billing problems
- Cost allocation tag issues
- Tag policy enforcement failures
- AI opt-out policy configuration
- Backup policy management
- Service integration failures
- Feature enablement issues
- Account migration between OUs
- Organization restructuring
- Account removal from organization

---

## Skill Structure

```
organizations-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-account-creation-failures.md
    ├── A2-account-closure.md
    ├── A3-account-suspension.md
    ├── B1-scp-evaluation.md
    ├── B2-scp-conflicts.md
    ├── B3-scp-inheritance.md
    ├── C1-ou-structure-issues.md
    ├── C2-ou-limits.md
    ├── D1-delegated-administrator.md
    ├── D2-trusted-access.md
    ├── E1-consolidated-billing.md
    ├── E2-cost-allocation.md
    ├── F1-tag-policies.md
    ├── F2-ai-opt-out.md
    ├── F3-backup-policies.md
    ├── G1-service-integration.md
    ├── G2-feature-enablement.md
    ├── H1-account-migration.md
    ├── H2-organization-restructuring.md
    ├── Z1-general-troubleshooting.md
    ├── organizations-guardrails.md
    └── organizations-hallucination-patterns.yaml
```

---

## Runbook Library (26 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Accounts** | A1–A3 | Account creation failures, account closure, account suspension |
| **B — SCPs** | B1–B3 | SCP evaluation, SCP conflicts, SCP inheritance |
| **C — OUs** | C1–C2 | OU structure issues, OU limits |
| **D — Delegation** | D1–D2 | Delegated administrator, trusted access |
| **E — Billing** | E1–E2 | Consolidated billing, cost allocation |
| **F — Policies** | F1–F3 | Tag policies, AI opt-out, backup policies |
| **G — Service Access** | G1–G2 | Service integration, feature enablement |
| **H — Migration** | H1–H2 | Account migration, organization restructuring |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/organizations-guardrails.md` covering management account SCP exemption, SCP deny-filter model, account email uniqueness, account closure recovery period, delegated admin limitations, OU nesting limits, SCP attachment limits, policy inheritance behavior, consolidated billing RI/SP sharing, account removal requirements, organization feature set immutability, and SCP size limits.

---

## Investigation Workflow

1. **Triage** — Describe organization, list accounts, check roots and OUs, list SCPs
2. **Deep Dive** — Examine policies for targets, describe specific policies, check delegated admins and service access
3. **Detailed** — Check effective policies, account creation status, handshakes, CloudTrail events

---

## Prerequisites

- AWS CLI v2 configured with management account credentials (most operations require management account)
- Permissions: `organizations:*`, `iam:GetRole`, `iam:GetPolicy`, `cloudtrail:LookupEvents`, `cloudwatch:GetMetricStatistics`, `ce:GetCostAndUsage` (for billing), `sso:*` (for IAM Identity Center)
- CloudTrail logging enabled (recommended)

---

## Usage Examples

```
# Describe the organization
aws organizations describe-organization

# List all accounts
aws organizations list-accounts

# List SCPs
aws organizations list-policies --filter SERVICE_CONTROL_POLICY

# Check SCPs attached to an OU
aws organizations list-policies-for-target --target-id ou-xxxx-xxxxxxxx --filter SERVICE_CONTROL_POLICY

# Check account creation status
aws organizations list-create-account-status --states FAILED

# List delegated administrators
aws organizations list-delegated-administrators

# Check enabled service integrations
aws organizations list-aws-service-access-for-organization
```

---

## License

MIT-0
