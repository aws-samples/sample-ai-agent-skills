---
name: lightsail-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Lightsail problems
  by analyzing instance issues, networking, databases, containers, storage,
  load balancers, CDN distributions, and following structured runbooks.
  Activate when: instance creation failures, instance connectivity issues,
  performance degradation, static IP problems, firewall configuration errors,
  DNS/domain issues, managed database failures, database connectivity problems,
  container service errors, container deployment failures, disk issues, snapshot
  failures, load balancer configuration problems, SSL/TLS certificate errors,
  CDN distribution issues, caching problems, or the user says something is
  wrong with Lightsail without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Lightsail permissions.
  Some operations require Lightsail console access.
---

# Amazon Lightsail Diagnostics

## When to use

Any Amazon Lightsail investigation where the console alone is insufficient — instance management, networking configuration, managed databases, container services, storage and snapshots, load balancers, CDN distributions, or DNS troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws lightsail get-instances --query 'instances[*].{Name:name,State:state.name,Blueprint:blueprintId,Bundle:bundleId,IP:publicIpAddress}'
version: "1.0.0"
last_updated: "2025-04-12"
aws lightsail get-instance --instance-name <instance-name>
aws lightsail get-instance-state --instance-name <instance-name>
aws lightsail get-instance-metric-data --instance-name <instance-name> --metric-name CPUUtilization --period 300 --start-time <start> --end-time <end> --unit Percent --statistics Average
```

### Step 2 — Domain deep dive

```
aws lightsail get-static-ips
aws lightsail get-instance-port-states --instance-name <instance-name>
aws lightsail get-domains
aws lightsail get-relational-databases
aws lightsail get-container-services
aws lightsail get-load-balancers
aws lightsail get-distributions
```

### Step 3 — Detailed investigation

```
aws lightsail get-instance-access-details --instance-name <instance-name>
aws lightsail get-relational-database-log-events --relational-database-name <db-name> --log-stream-name error
aws lightsail get-container-log --service-name <service-name> --container-name <container-name>
aws lightsail get-load-balancer-metric-data --load-balancer-name <lb-name> --metric-name HTTPCode_Target_5XX_Count --period 300 --start-time <start> --end-time <end> --unit Count --statistics Sum
```

Read `references/lightsail-guardrails.md` before concluding on any Lightsail issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `get-instance` | Check instance configuration, state, networking |
| `get-instance-port-states` | Check firewall rules |
| `get-static-ips` | Check static IP allocation and attachment |
| `get-relational-databases` | Check managed database status |
| `get-container-services` | Check container service configuration |
| `get-load-balancers` | Check load balancer health and configuration |
| `get-distributions` | Check CDN distribution status |
| `get-instance-metric-data` | Monitor instance performance metrics |

## Gotchas: Amazon Lightsail

- Lightsail instances have a built-in firewall that is separate from VPC security groups. The Lightsail firewall is configured per-instance and controls inbound traffic only. By default, SSH (22) and HTTP (80) are open. You must explicitly open additional ports. The firewall is NOT the same as VPC security groups.
- Static IPs are free only when attached to a running instance. Unattached static IPs or static IPs on stopped instances incur charges. Each account can have up to 5 static IPs per region. Static IPs persist across instance stop/start cycles.
- Lightsail managed databases are NOT the same as RDS. They have different feature sets, backup options, and scaling capabilities. Lightsail databases support MySQL and PostgreSQL only. They do not support Multi-AZ, read replicas, or IAM authentication.
- Container services have specific deployment size limits. Nano (0.25 vCPU, 512 MB), Micro (0.5 vCPU, 1 GB), Small (1 vCPU, 2 GB), Medium (2 vCPU, 4 GB), Large (4 vCPU, 8 GB), XLarge (8 vCPU, 16 GB). Scale is 1-20 nodes. Container images must be pushed to Lightsail container registry.
- Lightsail instances can be connected to VPC resources via VPC peering. Peering must be enabled in the Lightsail console. Once peered, Lightsail instances can access VPC resources using private IPs. Peering is one-way: VPC resources cannot initiate connections to Lightsail instances.
- Snapshots are the primary backup mechanism. Instance snapshots capture the entire instance (OS, data, configuration). Disk snapshots capture additional attached disks. Snapshots can be used to create new instances or restore. Automatic snapshots can be enabled per-instance.
- Lightsail DNS zones are free but have limitations. Each DNS zone supports up to 6 domain name servers. Lightsail DNS does not support all Route 53 record types. For advanced DNS features, use Route 53 instead.

## Anti-hallucination rules

1. Always cite specific instance names, IPs, or API responses as evidence.
2. Lightsail firewall is NOT VPC security groups. Never confuse the two.
3. Lightsail databases are NOT RDS. Never suggest RDS-specific features.
4. Container services have fixed size tiers. Never suggest custom CPU/memory.
5. Static IPs cost money when unattached. Never claim they are always free.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 20 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Instance | A1–A3 | Creation failures, connectivity, performance |
| B — Networking | B1–B3 | Static IP, firewall, DNS/domains |
| C — Database | C1–C2 | Managed database issues, connectivity |
| D — Container | D1–D2 | Container service, deployment |
| E — Storage | E1–E2 | Disk issues, snapshots |
| F — Load Balancer | F1–F2 | LB config, SSL/TLS |
| G — CDN | G1–G2 | Distribution issues, caching |
| Z — Catch-All | Z1 | General troubleshooting |
