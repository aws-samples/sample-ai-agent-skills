# AWS Cloud Map Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Cloud Map problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Cloud Map when the console alone isn't enough — namespace management, service registration debugging, instance health, DNS resolution, ECS/EKS service discovery integration, and IAM permission issues.

### Activate When

- DNS or HTTP namespace creation failures
- Service registration or configuration issues
- Health check failures or misconfiguration
- Instance registration or deregistration problems
- DNS resolution failures for Cloud Map services
- TTL configuration causing stale discovery
- ECS service discovery not working
- EKS service discovery integration issues
- IAM permission errors for Cloud Map operations
- VPC association problems for private namespaces

---

## Skill Structure

```
cloudmap-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-dns-namespace.md
    ├── A2-http-namespace.md
    ├── B1-service-registration.md
    ├── B2-health-checks.md
    ├── C1-instance-registration.md
    ├── C2-instance-deregistration.md
    ├── D1-dns-resolution.md
    ├── D2-ttl.md
    ├── E1-ecs-service-discovery.md
    ├── E2-eks-integration.md
    ├── F1-iam-permissions.md
    ├── F2-vpc-association.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (18 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Namespace** | A1–A2 | DNS namespace, HTTP namespace |
| **B — Service** | B1–B2 | Service registration, health checks |
| **C — Instance** | C1–C2 | Instance registration, deregistration |
| **D — DNS** | D1–D2 | DNS resolution, TTL |
| **E — Integration** | E1–E2 | ECS service discovery, EKS integration |
| **F — Security** | F1–F2 | IAM permissions, VPC association |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering namespace type differences, DNS record management, HTTP namespace API-only discovery, health check types, custom health check ownership, ECS integration boundaries, DNS TTL impact, instance attributes scope, namespace deletion ordering, VPC association requirements, eventual consistency, and service quota limits.

---

## Investigation Workflow

1. **Triage** — List namespaces, services, instances; check namespace type and status
2. **Deep Dive** — Check instance health, discover instances, examine DNS records
3. **Detailed** — Review CloudTrail events, check operations, verify Route 53 health checks

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `servicediscovery:*`, `route53:ListHostedZones`, `route53:ListResourceRecordSets`, `route53:GetHealthCheckStatus`, `ec2:DescribeVpcs`, `cloudtrail:LookupEvents`

---

## Usage Examples

```
# List all namespaces
aws servicediscovery list-namespaces

# Discover instances
aws servicediscovery discover-instances --namespace-name my-namespace --service-name my-service

# Check instance health
aws servicediscovery get-instances-health-status --service-id srv-1234567890

# Check DNS records
aws route53 list-resource-record-sets --hosted-zone-id Z1234567890
```

---

## License

MIT-0
