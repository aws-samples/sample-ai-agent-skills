# Amazon Lightsail Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Lightsail problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Lightsail when the console alone isn't enough — instance creation and connectivity, networking with static IPs and firewalls, DNS and domain management, managed databases, container services, storage and snapshots, load balancers with SSL/TLS, and CDN distributions.

### Activate When

- Instance creation or boot failures
- Instance connectivity or SSH issues
- Instance performance degradation
- Static IP allocation or attachment problems
- Firewall rule configuration errors
- DNS zone or domain resolution issues
- Managed database creation or access failures
- Database connectivity problems
- Container service deployment errors
- Container health check failures
- Disk attachment or capacity issues
- Snapshot creation or restore failures
- Load balancer configuration problems
- SSL/TLS certificate errors on load balancers
- CDN distribution creation or update issues
- CDN caching behavior problems
- General Lightsail errors without clear symptoms

---

## Skill Structure

```
lightsail-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-creation-failures.md
    ├── A2-connectivity.md
    ├── A3-performance.md
    ├── B1-static-ip.md
    ├── B2-firewall.md
    ├── B3-dns-domains.md
    ├── C1-managed-database.md
    ├── C2-database-connectivity.md
    ├── D1-container-service.md
    ├── D2-container-deployment.md
    ├── E1-disk-issues.md
    ├── E2-snapshots.md
    ├── F1-lb-config.md
    ├── F2-ssl-tls.md
    ├── G1-distribution-issues.md
    ├── G2-caching.md
    ├── Z1-general-troubleshooting.md
    ├── lightsail-guardrails.md
    └── lightsail-hallucination-patterns.yaml
```

---

## Runbook Library (20 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Instance** | A1–A3 | Creation failures, connectivity, performance |
| **B — Networking** | B1–B3 | Static IP, firewall, DNS/domains |
| **C — Database** | C1–C2 | Managed database issues, connectivity |
| **D — Container** | D1–D2 | Container service, deployment |
| **E — Storage** | E1–E2 | Disk issues, snapshots |
| **F — Load Balancer** | F1–F2 | LB config, SSL/TLS |
| **G — CDN** | G1–G2 | Distribution issues, caching |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/lightsail-guardrails.md` covering Lightsail firewall vs VPC security groups, static IP billing, managed database limitations vs RDS, container service size tiers, VPC peering behavior, snapshot mechanisms, DNS zone limitations, instance plan limits, bundle migration constraints, automatic snapshot scheduling, Lightsail vs EC2 feature differences, and load balancer health check configuration.

---

## Investigation Workflow

1. **Triage** — Check instance state, networking, firewall rules, resource status
2. **Deep Dive** — Examine specific resource configuration, metrics, logs
3. **Detailed** — Review access details, database logs, container logs, LB metrics

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `lightsail:Get*`, `lightsail:Describe*`
- Lightsail console access for some operations
- SSH key pair for instance access (optional)

---

## Usage Examples

```
# List all instances
aws lightsail get-instances

# Check instance firewall rules
aws lightsail get-instance-port-states --instance-name my-instance

# Check managed database status
aws lightsail get-relational-databases

# Check container service
aws lightsail get-container-services --service-name my-service

# Check load balancer
aws lightsail get-load-balancers
```

---

## License

MIT-0
