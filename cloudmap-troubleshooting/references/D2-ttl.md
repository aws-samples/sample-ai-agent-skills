---
title: "D2 — Cloud Map DNS TTL Issues"
description: "Diagnose TTL-related discovery latency problems"
status: active
severity: LOW
triggers:
  - "TTL"
  - "stale DNS"
  - "DNS cache"
  - "slow discovery"
  - "old IP address"
owner: devops-agent
objective: "Identify and resolve TTL configuration issues affecting discovery latency"
context: "DNS TTL determines how long clients cache DNS responses. High TTL causes slow discovery of changes. Low TTL increases DNS query volume. TTL is configured per-service in Cloud Map."
---

## Phase 1 — Triage

MUST:
- Check service DNS TTL: `aws servicediscovery get-service --id <service-id> --query 'Service.DnsConfig.DnsRecords[*].TTL'`
- Verify current DNS records: `aws route53 list-resource-record-sets --hosted-zone-id <zone-id> --query 'ResourceRecordSets[?Name==\`<service>.<namespace>.\`]'`
- Check if clients are caching beyond TTL (application-level caching)

SHOULD:
- Compare TTL with deployment frequency
- Check if DNS resolver is respecting TTL
- Verify no intermediate caching layers (corporate DNS, etc.)

MAY:
- Monitor DNS query volume to assess TTL impact
- Check client-side DNS caching behavior

## Phase 2 — Remediate

MUST:
- Set TTL appropriate for the use case (lower for frequent changes, higher for stable services)
- Note: TTL can only be set at service creation, not updated after

SHOULD:
- Use 10-60 second TTL for services with frequent instance changes
- Use 60-300 second TTL for stable services
- Document TTL decisions and rationale

MAY:
- Implement client-side service discovery caching with shorter TTL
- Use API discovery (DiscoverInstances) for real-time needs instead of DNS

## Common Issues

- symptoms: "Clients connecting to old IP after instance replacement"
  diagnosis: "DNS TTL too high — clients caching stale records."
  resolution: "Create a new service with lower TTL (TTL cannot be updated on existing services)."

- symptoms: "High DNS query volume"
  diagnosis: "TTL set too low causing frequent DNS lookups."
  resolution: "Increase TTL if service instances are stable."

- symptoms: "TTL set correctly but clients still use old IP"
  diagnosis: "Application-level DNS caching or JVM DNS cache."
  resolution: "Configure application DNS cache TTL (e.g., JVM networkaddress.cache.ttl)."

## Output Format

```yaml
root_cause: "ttl_issue — <specific_cause>"
evidence:
  - type: ttl_config
    content: "<service TTL setting>"
  - type: dns_records
    content: "<current DNS records and TTL>"
severity: LOW
mitigation:
  immediate: "Create new service with appropriate TTL if change needed"
  long_term: "Standardize TTL settings based on service stability profiles"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Set TTL appropriate for use case (at service creation) | YELLOW |
| Use 10-60 second TTL for frequent changes | GREEN |
| Use 60-300 second TTL for stable services | GREEN |
| Implement client-side caching with shorter TTL | GREEN |
| Use API discovery for real-time needs | GREEN |

## Escalation Conditions

- Fix requires recreating production services to change TTL
- Fix involves modifying client-side DNS caching behavior

## Data Sensitivity

- **LOW**: TTL configurations affect discovery latency but do not expose sensitive data
- TTL patterns may reveal deployment frequency and service stability

## Prohibited Actions

- NEVER suggest deleting namespaces with active service registrations

## Phase 3 — Rollback

1. TTL cannot be changed on existing services — must create new service with correct TTL
2. If a new service was created, migrate instances from old service before deleting it
3. If client-side caching was modified, restore previous cache settings
4. Verify rollback by confirming DNS TTL and resolution behavior match expected state

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
