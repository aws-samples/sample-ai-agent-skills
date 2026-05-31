---
title: "B1 — Cloud Map Service Registration Issues"
description: "Diagnose service creation and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "service registration"
  - "create service"
  - "service configuration"
  - "DNS record type"
owner: devops-agent
objective: "Identify and resolve service registration issues in Cloud Map"
context: "Cloud Map services define how instances are discovered — DNS record type (A, SRV, CNAME), health check configuration, and namespace association. Service configuration affects how clients discover instances."
---

## Phase 1 — Triage

MUST:
- Check service details: `aws servicediscovery get-service --id <service-id>`
- Verify DNS record configuration: check DnsConfig in service spec
- Check health check configuration: check HealthCheckConfig or HealthCheckCustomConfig
- List instances: `aws servicediscovery list-instances --service-id <service-id>`
- Check operation status if creation pending: `aws servicediscovery get-operation --operation-id <operation-id>`

SHOULD:
- Verify DNS record type matches use case (A for IP, SRV for port+IP, CNAME for alias)
- Check routing policy (MULTIVALUE or WEIGHTED)
- Verify namespace association

MAY:
- Check service quotas for services per namespace
- Compare with working service configurations

## Phase 2 — Remediate

MUST:
- Configure correct DNS record type for the use case
- Set appropriate TTL for the DNS records
- Configure health checks if instance health monitoring is needed

SHOULD:
- Use SRV records when port information is needed
- Use MULTIVALUE routing for load distribution
- Set health check thresholds appropriately

MAY:
- Use tags for service organization
- Implement service creation via IaC

## Common Issues

- symptoms: "Service creation fails"
  diagnosis: "Invalid DNS configuration or namespace doesn't support the record type."
  resolution: "Verify DNS record type is valid for the namespace type."

- symptoms: "Instances registered but not discoverable via DNS"
  diagnosis: "DNS record type mismatch or TTL too high."
  resolution: "Check DNS record type and TTL. Verify Route 53 records are created."

- symptoms: "Service health check not working"
  diagnosis: "Health check type incompatible with endpoint (Route 53 for private IPs)."
  resolution: "Use custom health checks for private endpoints."

## Output Format

```yaml
root_cause: "service_registration_issue — <specific_cause>"
evidence:
  - type: service_config
    content: "<DNS config, health check, routing policy>"
  - type: instances
    content: "<registered instances>"
  - type: dns_records
    content: "<Route 53 records>"
severity: MEDIUM
mitigation:
  immediate: "Fix service configuration"
  long_term: "Standardize service configurations across namespaces"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Configure correct DNS record type | YELLOW |
| Set appropriate TTL for DNS records | GREEN |
| Configure health checks | GREEN |
| Use SRV records when port information is needed | GREEN |
| Use MULTIVALUE routing for load distribution | GREEN |

## Escalation Conditions

- Fix requires modifying production service configurations
- Fix involves changing DNS record types or routing policies for active services

## Data Sensitivity

- **MEDIUM**: service configurations expose discovery topology and health check endpoints
- DNS record types and routing policies reveal service architecture

## Prohibited Actions

- NEVER suggest deleting namespaces with active service registrations

## Phase 3 — Rollback

1. If service was created with wrong configuration, delete and recreate (only if no active instances)
2. If health check configuration was changed, restore previous settings
3. Note: DNS record type and TTL cannot be changed after service creation — must recreate
4. Verify rollback by confirming service configuration matches expected state

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
