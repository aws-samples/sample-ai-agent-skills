# AWS Verified Access Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Verified Access problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Verified Access when the console alone isn't enough — instance creation and trust provider attachment, group policy configuration and evaluation, endpoint creation and DNS setup, identity and device trust providers, Cedar policy language debugging, access logging and analysis, and VPC networking configuration.

### Activate When

- Verified Access instance creation failures
- Trust provider configuration or attachment errors
- Group policy not evaluating as expected
- Policy denying legitimate access
- Endpoint creation or status failures
- DNS not resolving to Verified Access endpoint
- Endpoint connectivity issues
- Identity trust provider (OIDC/IAM Identity Center) errors
- Device trust provider integration problems
- Cedar policy syntax errors
- Policy debugging and testing difficulties
- Access logs not appearing
- CloudWatch integration issues
- VPC configuration problems for endpoints
- Security group blocking Verified Access traffic
- General Verified Access errors without clear symptoms

---

## Skill Structure

```
verifiedaccess-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-instance-creation.md
    ├── A2-trust-provider-config.md
    ├── B1-group-policies.md
    ├── B2-policy-evaluation.md
    ├── C1-endpoint-creation.md
    ├── C2-endpoint-dns.md
    ├── C3-endpoint-connectivity.md
    ├── D1-identity-trust-provider.md
    ├── D2-device-trust-provider.md
    ├── E1-cedar-policy-language.md
    ├── E2-policy-debugging.md
    ├── F1-access-logs.md
    ├── F2-cloudwatch-integration.md
    ├── G1-vpc-config.md
    ├── G2-security-groups.md
    ├── Z1-general-troubleshooting.md
    ├── verifiedaccess-guardrails.md
    └── verifiedaccess-hallucination-patterns.yaml
```

---

## Runbook Library (20 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Instance** | A1–A2 | Instance creation, trust provider config |
| **B — Group** | B1–B2 | Group policies, policy evaluation |
| **C — Endpoint** | C1–C3 | Endpoint creation, DNS, connectivity |
| **D — Trust** | D1–D2 | Identity trust provider, device trust provider |
| **E — Policy** | E1–E2 | Cedar policy language, policy debugging |
| **F — Logging** | F1–F2 | Access logs, CloudWatch integration |
| **G — Networking** | G1–G2 | VPC config, security groups |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/verifiedaccess-guardrails.md` covering Cedar vs IAM policy distinction, trust provider types and context, endpoint-to-group relationship, application-layer vs network-layer operation, logging requirements, DNS configuration, policy evaluation order, group inheritance, endpoint types, security group requirements, trust provider attachment limits, and Cedar policy syntax rules.

---

## Investigation Workflow

1. **Triage** — Check instance, groups, endpoints, trust providers status
2. **Deep Dive** — Examine policies, endpoint configuration, trust provider details
3. **Detailed** — Review access logs, CloudTrail events, policy evaluation results

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `ec2:DescribeVerifiedAccess*`, `ec2:GetVerifiedAccessGroupPolicy`, `logs:FilterLogEvents`, `cloudtrail:LookupEvents`
- Access to identity provider admin console for trust provider troubleshooting
- DNS management access for endpoint DNS configuration

---

## Usage Examples

```
# List Verified Access instances
aws ec2 describe-verified-access-instances

# Check group policy
aws ec2 get-verified-access-group-policy --verified-access-group-id <group-id>

# Check endpoint status
aws ec2 describe-verified-access-endpoints --verified-access-endpoint-ids <endpoint-id>

# Check logging configuration
aws ec2 describe-verified-access-instance-logging-configurations --verified-access-instance-ids <instance-id>

# Review access logs
aws logs filter-log-events --log-group-name <log-group> --filter-pattern "DENY"
```

---

## License

MIT-0
