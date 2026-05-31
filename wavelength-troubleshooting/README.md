# Wavelength Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Wavelength problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Wavelength when the console alone isn't enough — Wavelength Zone deployment, carrier gateway configuration, compute instances, carrier IP management, VPC networking, ultra-low latency optimization, and Region integration.

### Activate When

- Wavelength Zone opt-in or availability issues
- Carrier gateway configuration problems
- EC2 instance launch failures in Wavelength Zones
- ECS/EKS deployment issues in Wavelength
- Carrier IP allocation or routing problems
- VPC subnet configuration for Wavelength
- High latency on carrier network path
- Throughput limitations
- Region connectivity from Wavelength
- S3 access from Wavelength Zones

---

## Skill Structure

```
wavelength-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-wavelength-zone.md
    ├── A2-carrier-gateway.md
    ├── B1-ec2-in-wavelength.md
    ├── B2-ecs-eks.md
    ├── C1-carrier-ip.md
    ├── C2-vpc-subnets.md
    ├── D1-latency.md
    ├── D2-throughput.md
    ├── E1-region-connectivity.md
    ├── E2-s3-access.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (16 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Zone** | A1–A2 | Wavelength zone, carrier gateway |
| **B — Compute** | B1–B2 | EC2 in Wavelength, ECS/EKS |
| **C — Networking** | C1–C2 | Carrier IP, VPC subnets |
| **D — Performance** | D1–D2 | Latency, throughput |
| **E — Integration** | E1–E2 | Region connectivity, S3 access |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering Wavelength Zone opt-in, carrier gateway requirements, carrier IP vs Elastic IP, instance type availability, no IGW/NAT support, EBS local storage, S3 access patterns, DNS resolution, Auto Scaling constraints, service availability limits, subnet configuration, and carrier network dependencies.

---

## Investigation Workflow

1. **Triage** — Verify Wavelength Zone opt-in, check carrier gateway, review instance status
2. **Deep Dive** — Examine routing, security groups, network interfaces, container orchestration
3. **Detailed** — Inspect CloudWatch metrics, CloudTrail events, VPC endpoints

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `ec2:Describe*`, `ec2:*CarrierGateway*`, `ecs:*`, `eks:*`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`
- Wavelength Zone opted-in for the account
- Carrier partner relationship (Verizon, Vodafone, etc.)

---

## Usage Examples

```
# List Wavelength Zones
aws ec2 describe-availability-zones --filters "Name=zone-type,Values=wavelength-zone" --all-availability-zones

# Check carrier gateway
aws ec2 describe-carrier-gateways

# List carrier IPs
aws ec2 describe-addresses --filters "Name=domain,Values=vpc" --query 'Addresses[?CarrierIp!=null]'

# Check instances in Wavelength Zone
aws ec2 describe-instances --filters "Name=availability-zone,Values=us-east-1-wl1-bos-wlz-1"
```

---

## License

MIT-0
