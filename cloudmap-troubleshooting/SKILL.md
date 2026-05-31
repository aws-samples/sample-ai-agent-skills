---
name: cloudmap-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Cloud Map problems by
  analyzing namespace configurations, service registrations, instance
  management, DNS resolution, TTL settings, ECS service discovery, EKS
  integration, IAM permissions, and VPC associations using structured
  runbooks. Activate when: DNS or HTTP namespace issues, service registration
  failures, health check problems, instance registration or deregistration
  issues, DNS resolution failures, TTL configuration problems, ECS service
  discovery not working, EKS integration issues, IAM permission errors, VPC
  association problems, or the user says something is wrong with Cloud Map
  without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with ServiceDiscovery, Route 53, EC2 (VPC),
  ECS, EKS, IAM, CloudTrail, and optionally CloudWatch permissions.
---

# AWS Cloud Map Diagnostics

## When to use

Any AWS Cloud Map investigation where the console alone is insufficient — namespace management, service registration debugging, instance health, DNS resolution, ECS/EKS service discovery integration, or IAM permission issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws servicediscovery list-namespaces
aws servicediscovery get-namespace --id <namespace-id>
aws servicediscovery list-services --filters Name=NAMESPACE_ID,Values=<namespace-id>
aws servicediscovery get-service --id <service-id>
aws servicediscovery list-instances --service-id <service-id>
```

### Step 2 — Domain deep dive

```
aws servicediscovery get-instance --service-id <service-id> --instance-id <instance-id>
aws servicediscovery get-instances-health-status --service-id <service-id>
aws servicediscovery discover-instances --namespace-name <namespace> --service-name <service>
aws route53 list-hosted-zones --query 'HostedZones[?Name==\`<namespace-name>.\`]'
aws route53 list-resource-record-sets --hosted-zone-id <zone-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=servicediscovery.amazonaws.com --max-results 20
aws servicediscovery get-operation --operation-id <operation-id>
aws route53 get-health-check-status --health-check-id <health-check-id>
```

Read `references/guardrails.md` before concluding on any Cloud Map issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `servicediscovery list-namespaces` | List all namespaces |
| `servicediscovery get-namespace` | Namespace details and type |
| `servicediscovery list-services` | Services in a namespace |
| `servicediscovery get-service` | Service configuration and health check |
| `servicediscovery list-instances` | Instances registered to a service |
| `servicediscovery discover-instances` | Query instances (API discovery) |
| `servicediscovery get-instances-health-status` | Instance health status |
| `route53 list-resource-record-sets` | DNS records for DNS namespaces |
| `route53 get-health-check-status` | Route 53 health check results |
| `cloudtrail lookup-events` | Recent Cloud Map API events |

## Gotchas: AWS Cloud Map

- Cloud Map has two namespace types: DNS (public or private) and HTTP. DNS namespaces create Route 53 hosted zones. HTTP namespaces use API-only discovery (no DNS records).
- DNS namespaces create Route 53 records automatically — when instances register, A or SRV records are created. When instances deregister, records are removed. Manual Route 53 changes can conflict.
- HTTP namespaces are API-only — no DNS records are created. Discovery uses the DiscoverInstances API. Useful when DNS is not needed or for non-IP-based discovery.
- Health checks are optional but recommended — Cloud Map supports Route 53 health checks (for public endpoints) and custom health checks (for private endpoints). Without health checks, unhealthy instances remain discoverable.
- Custom health checks require application-managed status — your application must call UpdateInstanceCustomHealthStatus to report health. Cloud Map does not automatically check health for custom health checks.
- ECS service discovery uses Cloud Map automatically — ECS creates and manages Cloud Map services and instances. Manual modification of ECS-managed Cloud Map resources can cause conflicts.
- DNS TTL affects discovery latency — lower TTL means faster discovery of changes but more DNS queries. Default TTL is 60 seconds. During deployments, high TTL causes stale DNS responses.
- Instance attributes are key-value pairs — used for API-based discovery filtering. Attributes are not reflected in DNS records.
- Namespace deletion requires all services to be deleted first — services must have all instances deregistered before deletion. Cascading deletion is not automatic.
- VPC association is required for private DNS namespaces — the Route 53 private hosted zone must be associated with the VPC where clients resolve DNS.
- Cloud Map operations are eventually consistent — registration and deregistration may take a few seconds to propagate. DNS changes depend on TTL.
- Service quotas limit instances per service and services per namespace — check quotas before large-scale deployments.

## Anti-hallucination rules

1. Always cite specific namespace configurations, service details, or API responses as evidence. Never guess at Cloud Map settings.
2. DNS namespaces create Route 53 records; HTTP namespaces do NOT. Never suggest DNS resolution for HTTP namespaces.
3. Custom health checks require application-managed status updates. Never suggest Cloud Map automatically checks custom health.
4. ECS-managed Cloud Map resources should not be manually modified. Never suggest direct modification of ECS-created services.
5. Private DNS namespaces require VPC association. Never assume DNS resolution works without VPC association.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 18 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Namespace | A1-A2 | DNS namespace, HTTP namespace |
| B — Service | B1-B2 | Service registration, health checks |
| C — Instance | C1-C2 | Instance registration, deregistration |
| D — DNS | D1-D2 | DNS resolution, TTL |
| E — Integration | E1-E2 | ECS service discovery, EKS integration |
| F — Security | F1-F2 | IAM permissions, VPC association |
| Z — Catch-All | Z1 | General troubleshooting |
