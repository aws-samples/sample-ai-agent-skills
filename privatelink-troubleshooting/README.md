# AWS PrivateLink / VPC Interface Endpoints Diagnostics Skill

Agent skill for investigating and troubleshooting AWS PrivateLink and VPC Interface Endpoint problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS PrivateLink when the console alone isn't enough — endpoint creation and status, DNS resolution, endpoint service configuration, NLB requirements, connection acceptance, connectivity through endpoints, security group configuration, cross-AZ traffic, private DNS, split-horizon DNS, cross-account access, endpoint policies, and performance optimization.

### Activate When

- Interface endpoint creation failures
- DNS not resolving to private IPs through endpoint
- Endpoint stuck in pending or failed state
- Endpoint service creation issues
- NLB not properly associated with endpoint service
- Connection acceptance stuck in pending
- Connectivity failures through PrivateLink endpoints
- Security groups blocking traffic to/from endpoint ENIs
- Cross-AZ data transfer cost concerns
- Private DNS not working for interface endpoints
- Split-horizon DNS configuration issues
- Cross-account endpoint access denied
- Endpoint policy blocking legitimate traffic
- Throughput limitations through endpoints
- Latency issues through PrivateLink
- General PrivateLink errors without clear symptoms

---

## Skill Structure

```
privatelink-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-creation-failures.md
    ├── A2-dns-resolution.md
    ├── A3-endpoint-status.md
    ├── B1-endpoint-service-creation.md
    ├── B2-nlb-requirements.md
    ├── B3-acceptance.md
    ├── C1-connection-failures.md
    ├── C2-security-groups.md
    ├── C3-cross-az.md
    ├── D1-private-dns.md
    ├── D2-split-horizon-dns.md
    ├── E1-cross-account-access.md
    ├── E2-endpoint-policies.md
    ├── F1-throughput.md
    ├── F2-latency.md
    ├── G1-endpoint-policies-security.md
    ├── G2-security-groups-audit.md
    ├── Z1-general-troubleshooting.md
    ├── privatelink-guardrails.md
    └── privatelink-hallucination-patterns.yaml
```

---

## Runbook Library (22 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Endpoint** | A1–A3 | Creation failures, DNS resolution, endpoint status |
| **B — Service** | B1–B3 | Endpoint service creation, NLB requirements, acceptance |
| **C — Connectivity** | C1–C3 | Connection failures, security groups, cross-AZ |
| **D — DNS** | D1–D2 | Private DNS, split-horizon DNS |
| **E — Cross-Account** | E1–E2 | Cross-account access, endpoint policies |
| **F — Performance** | F1–F2 | Throughput, latency |
| **G — Security** | G1–G2 | Endpoint policies, security groups |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/privatelink-guardrails.md` covering interface-vs-gateway endpoint distinction, NLB requirement for endpoint services, VPC DNS attribute requirements for private DNS, endpoint policy vs IAM policy distinction, cross-AZ data transfer costs, one-endpoint-per-VPC-per-service limit, security group configuration on endpoint ENIs, connection acceptance behavior, DNS resolution behavior with/without private DNS, subnet and AZ placement, endpoint service permissions, and VPC endpoint lifecycle states.

---

## Investigation Workflow

1. **Triage** — Check endpoint status, DNS entries, security groups, VPC DNS settings
2. **Deep Dive** — Examine ENIs, security group rules, connection status, endpoint service config
3. **Detailed** — Review CloudTrail events, VPC Flow Logs, NLB health, endpoint policies

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `ec2:DescribeVpcEndpoints`, `ec2:DescribeVpcEndpointServices`, `ec2:DescribeNetworkInterfaces`, `ec2:DescribeSecurityGroups`, `ec2:DescribeVpcs`, `elasticloadbalancing:DescribeLoadBalancers`, `cloudtrail:LookupEvents`
- VPC Flow Logs enabled for traffic analysis (optional but recommended)
- Access to both consumer and provider accounts for cross-account issues

---

## Usage Examples

```
# List all endpoints in a VPC
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=vpc-12345678

# Check endpoint status and DNS
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids vpce-12345678

# Check endpoint ENIs and private IPs
aws ec2 describe-network-interfaces --filters Name=vpc-endpoint-id,Values=vpce-12345678

# Check endpoint service configuration
aws ec2 describe-vpc-endpoint-service-configurations --service-ids vpce-svc-12345678

# Check connection acceptance status
aws ec2 describe-vpc-endpoint-connections --filters Name=vpc-endpoint-id,Values=vpce-12345678

# Check VPC DNS settings
aws ec2 describe-vpc-attribute --vpc-id vpc-12345678 --attribute enableDnsSupport
aws ec2 describe-vpc-attribute --vpc-id vpc-12345678 --attribute enableDnsHostnames
```

---

## License

MIT-0
