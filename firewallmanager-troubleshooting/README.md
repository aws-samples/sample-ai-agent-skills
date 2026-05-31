# AWS Firewall Manager Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Firewall Manager problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Firewall Manager when the console alone isn't enough — policy creation and compliance, auto-remediation, WAF policy and rule group management, Shield Advanced protection, security group policies, Network Firewall deployment, organization admin configuration, member account compliance, DNS Firewall policies, and domain list management.

### Activate When

- Firewall Manager policy creation failures
- Resources not compliant with FMS policies
- Auto-remediation not working as expected
- WAF policy not applying to resources
- WAF rule group conflicts or errors
- Shield Advanced policy issues
- DDoS protection gaps in member accounts
- Security group policy violations
- Security group audit findings
- Network Firewall policy deployment failures
- Network Firewall endpoints not created
- FMS admin account configuration issues
- Member account compliance problems
- DNS Firewall policy not blocking domains
- Domain list configuration errors
- General Firewall Manager errors without clear symptoms

---

## Skill Structure

```
firewallmanager-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-policy-creation.md
    ├── A2-policy-compliance.md
    ├── A3-auto-remediation.md
    ├── B1-waf-policy-issues.md
    ├── B2-rule-group-management.md
    ├── C1-shield-advanced-policy.md
    ├── C2-ddos-protection.md
    ├── D1-security-group-policy.md
    ├── D2-audit-policy.md
    ├── E1-network-firewall-policy.md
    ├── E2-network-firewall-deployment.md
    ├── F1-admin-account.md
    ├── F2-member-compliance.md
    ├── G1-dns-firewall-policy.md
    ├── G2-domain-lists.md
    ├── Z1-general-troubleshooting.md
    ├── firewallmanager-guardrails.md
    └── firewallmanager-hallucination-patterns.yaml
```

---

## Runbook Library (20 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Policy** | A1–A3 | Policy creation, policy compliance, auto-remediation |
| **B — WAF** | B1–B2 | WAF policy issues, rule group management |
| **C — Shield** | C1–C2 | Shield Advanced policy, DDoS protection |
| **D — Security Groups** | D1–D2 | Security group policy, audit policy |
| **E — Network Firewall** | E1–E2 | Network Firewall policy, deployment |
| **F — Organization** | F1–F2 | Admin account, member compliance |
| **G — DNS Firewall** | G1–G2 | DNS Firewall policy, domain lists |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/firewallmanager-guardrails.md` covering Organizations requirement, FMS administrator account designation, auto-remediation limitations, WAF web ACL management, Shield Advanced subscription costs, security group policy types, Network Firewall deployment behavior, DNS Firewall domain list management, policy scope configuration, cross-account policy enforcement, compliance evaluation timing, and FMS-managed resource naming conventions.

---

## Investigation Workflow

1. **Triage** — Check FMS admin account, list policies, verify compliance status
2. **Deep Dive** — Examine policy configuration, violation details, member account status
3. **Detailed** — Review CloudTrail events, WAF/Shield/Network Firewall resources, remediation actions

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `fms:*`, `wafv2:List*`, `wafv2:Get*`, `shield:Describe*`, `network-firewall:List*`, `ec2:DescribeSecurityGroups`, `organizations:Describe*`, `cloudtrail:LookupEvents`
- AWS Organizations with all features enabled
- FMS administrator account designated

---

## Usage Examples

```
# Check FMS admin account
aws fms get-admin-account

# List all FMS policies
aws fms list-policies

# Check compliance for a policy
aws fms list-compliance-status --policy-id p-12345678

# Get violation details
aws fms get-violation-details --policy-id p-12345678 --member-account 123456789012 --resource-id sg-12345678 --resource-type AWS::EC2::SecurityGroup

# Get policy details
aws fms get-policy --policy-id p-12345678
```

---

## License

MIT-0
