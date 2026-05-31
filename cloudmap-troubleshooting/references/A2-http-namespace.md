---
title: "A2 — Cloud Map HTTP Namespace Issues"
description: "Diagnose HTTP namespace creation and API discovery problems"
status: active
severity: MEDIUM
triggers:
  - "HTTP namespace"
  - "API discovery"
  - "DiscoverInstances"
  - "no DNS"
owner: devops-agent
objective: "Identify and resolve HTTP namespace issues in Cloud Map"
context: "HTTP namespaces are API-only — no DNS records or Route 53 hosted zones are created. Discovery uses the DiscoverInstances API. Useful for non-IP-based discovery or attribute-based filtering."
---

## Phase 1 — Triage

MUST:
- Check namespace type: `aws servicediscovery get-namespace --id <namespace-id> --query 'Namespace.Type'`
- List services in namespace: `aws servicediscovery list-services --filters Name=NAMESPACE_ID,Values=<namespace-id>`
- Test API discovery: `aws servicediscovery discover-instances --namespace-name <namespace> --service-name <service>`
- Verify instances are registered: `aws servicediscovery list-instances --service-id <service-id>`

SHOULD:
- Check instance attributes for filtering
- Verify health status of instances: `aws servicediscovery get-instances-health-status --service-id <service-id>`
- Check IAM permissions for DiscoverInstances

MAY:
- Test attribute-based filtering: `aws servicediscovery discover-instances --namespace-name <namespace> --service-name <service> --query-parameters key=value`
- Check API call throttling

## Phase 2 — Remediate

MUST:
- Use DiscoverInstances API for HTTP namespace discovery (not DNS)
- Register instances with correct attributes
- Grant servicediscovery:DiscoverInstances permission to clients

SHOULD:
- Set meaningful instance attributes for filtering
- Implement health reporting for custom health checks
- Document the API discovery pattern for consumers

MAY:
- Cache DiscoverInstances results with appropriate TTL
- Implement client-side load balancing across discovered instances

## Common Issues

- symptoms: "DNS resolution fails for HTTP namespace service"
  diagnosis: "HTTP namespaces don't create DNS records — DNS resolution is not supported."
  resolution: "Use DiscoverInstances API instead of DNS for HTTP namespaces."

- symptoms: "DiscoverInstances returns empty results"
  diagnosis: "No healthy instances registered or health filter excluding all instances."
  resolution: "Check instance registration and health status."

- symptoms: "Attribute filtering not working"
  diagnosis: "Instance attributes don't match query parameters."
  resolution: "Verify instance attributes match the query parameter keys and values."

## Output Format

```yaml
root_cause: "http_namespace_issue — <specific_cause>"
evidence:
  - type: namespace_type
    content: "<HTTP namespace confirmation>"
  - type: instances
    content: "<registered instances and attributes>"
  - type: discovery_result
    content: "<DiscoverInstances API response>"
severity: MEDIUM
mitigation:
  immediate: "Fix instance registration or discovery query"
  long_term: "Document API discovery patterns for all consumers"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Use DiscoverInstances API for HTTP namespace discovery | GREEN |
| Register instances with correct attributes | GREEN |
| Grant servicediscovery:DiscoverInstances permission | YELLOW |
| Set meaningful instance attributes | GREEN |
| Implement health reporting for custom health checks | GREEN |
| Cache DiscoverInstances results | GREEN |

## Escalation Conditions

- Fix requires modifying production HTTP namespace configurations
- Fix involves changing instance attributes used by production consumers

## Data Sensitivity

- **MEDIUM**: HTTP namespace configurations expose service endpoints and discovery attributes
- Instance attributes may contain environment-specific connection details

## Prohibited Actions

- NEVER suggest deleting namespaces with active service registrations

## Phase 3 — Rollback

1. If instance attributes were changed, restore previous attribute values
2. If permissions were modified, revert IAM policy changes
3. If health reporting was implemented, verify it does not affect existing health status
4. Verify rollback by confirming `aws servicediscovery discover-instances` returns expected results

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
