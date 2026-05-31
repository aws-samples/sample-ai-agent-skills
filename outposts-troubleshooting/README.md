# Outposts Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Outposts problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Outposts when the console alone isn't enough — rack connectivity, compute capacity, local storage, networking with local gateways, service link issues, hybrid connectivity, management operations, and security configuration.

### Activate When

- Outpost rack connectivity failures
- Capacity exhaustion or instance launch failures
- Hardware component failures
- EC2 instance issues on Outposts
- ECS task failures on Outposts
- EBS local volume problems
- S3 on Outposts access or configuration issues
- Local gateway routing problems
- Service link disconnections
- Outpost update failures
- Monitoring gaps or missing metrics
- Region connectivity loss
- Failover behavior issues
- Encryption configuration problems
- IAM permission issues on Outposts

---

## Skill Structure

```
outposts-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-connectivity.md
    ├── A2-capacity.md
    ├── A3-hardware-failures.md
    ├── B1-ec2-on-outposts.md
    ├── B2-ecs-on-outposts.md
    ├── C1-ebs-local.md
    ├── C2-s3-on-outposts.md
    ├── D1-local-gateway.md
    ├── D2-service-link.md
    ├── E1-updates.md
    ├── E2-monitoring.md
    ├── F1-region-connectivity.md
    ├── F2-failover.md
    ├── G1-encryption.md
    ├── G2-iam.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (20 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Rack** | A1–A3 | Connectivity, capacity, hardware failures |
| **B — Compute** | B1–B2 | EC2 on Outposts, ECS on Outposts |
| **C — Storage** | C1–C2 | EBS local, S3 on Outposts |
| **D — Networking** | D1–D2 | Local gateway, service link |
| **E — Management** | E1–E2 | Updates, monitoring |
| **F — Hybrid** | F1–F2 | Region connectivity, failover |
| **G — Security** | G1–G2 | Encryption, IAM |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering service link behavior, capacity limits, S3 access point requirements, local gateway routing, EBS local storage, instance type availability, subnet configuration, DNS resolution, shared Outposts, update requirements, CoIP pool management, and rack power/network dependencies.

---

## Investigation Workflow

1. **Triage** — Check Outpost connectivity status, capacity metrics, and hardware asset health
2. **Deep Dive** — Examine local gateway routes, service link, storage volumes, compute instances
3. **Detailed** — Inspect CloudTrail events, network interfaces, subnet configurations

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `outposts:*`, `ec2:Describe*`, `s3:*`, `ecs:*`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`
- Physical Outpost rack installed and powered on
- Service link connectivity to parent AWS Region

---

## Usage Examples

```
# List all Outposts
aws outposts list-outposts

# Check Outpost details
aws outposts get-outpost --outpost-id op-1234567890abcdef0

# Check available instance types
aws outposts get-outpost-instance-types --outpost-id op-1234567890abcdef0

# List EC2 instances on Outpost
aws ec2 describe-instances --filters "Name=outpost-arn,Values=arn:aws:outposts:us-east-1:123456789012:outpost/op-1234567890abcdef0"

# Check connectivity status
aws cloudwatch get-metric-statistics --namespace AWS/Outposts \
  --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=op-1234567890abcdef0 \
  --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 300 --statistics Average
```

---

## License

MIT-0
