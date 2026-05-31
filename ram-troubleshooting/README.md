# AWS Resource Access Manager (RAM) Diagnostics Skill

Agent skill for investigating and troubleshooting AWS RAM problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS RAM when the console alone isn't enough — resource share creation and management, invitation handling, cross-account resource access, organization sharing, permission management, and resource visibility issues.

### Activate When

- Resource share creation failures
- Invitation not received or expired
- Invitation acceptance errors
- Unsupported resource type sharing attempts
- Cross-account resource sharing failures
- Shared resources not accessible
- Organization sharing not working
- Auto-sharing configuration problems
- Managed permission errors
- Custom permission creation issues
- Access denied when accessing shared resources
- Shared resources not visible in participant account
- General RAM errors without clear symptoms

---

## Skill Structure

```
ram-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-resource-share-creation.md
    ├── A2-invitation.md
    ├── A3-acceptance.md
    ├── B1-supported-resources.md
    ├── B2-sharing-failures.md
    ├── B3-cross-account.md
    ├── C1-organization-sharing.md
    ├── C2-auto-sharing.md
    ├── D1-managed-permissions.md
    ├── D2-custom-permissions.md
    ├── E1-access-denied.md
    ├── E2-resource-not-visible.md
    ├── Z1-general-troubleshooting.md
    ├── ram-guardrails.md
    └── ram-hallucination-patterns.yaml
```

---

## Runbook Library (18 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Sharing** | A1–A3 | Resource share creation, invitation, acceptance |
| **B — Resources** | B1–B3 | Supported resources, sharing failures, cross-account |
| **C — Organization** | C1–C2 | Organization sharing, auto-sharing |
| **D — Permissions** | D1–D2 | Managed permissions, custom permissions |
| **E — Troubleshooting** | E1–E2 | Access denied, resource not visible |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/ram-guardrails.md` covering organization vs external sharing, supported resource types, invitation requirements, RAM permissions vs IAM policies, shared VPC subnet limitations, resource share lifecycle states, organization sharing enablement, cross-region restrictions, resource share deletion impact, principal types, permission versioning, and resource ownership.

---

## Investigation Workflow

1. **Triage** — Check resource shares, invitations, principals, organization status
2. **Deep Dive** — Examine permissions, resource associations, invitation details
3. **Detailed** — Review CloudTrail events, pending invitations, association status

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `ram:*`, `organizations:Describe*`, `iam:SimulatePrincipalPolicy`, `cloudtrail:LookupEvents`
- Organization management account access for organization sharing setup
- Both owner and participant account access for cross-account troubleshooting

---

## Usage Examples

```
# List resource shares you own
aws ram get-resource-shares --resource-owner SELF

# Check pending invitations
aws ram get-resource-share-invitations

# List shareable resource types
aws ram list-resource-types

# Accept an invitation
aws ram accept-resource-share-invitation --resource-share-invitation-arn <arn>

# Enable organization sharing
aws ram enable-sharing-with-aws-organization
```

---

## License

MIT-0
