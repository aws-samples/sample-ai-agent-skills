---
title: "C2 — Service Discovery"
description: "Diagnose service discovery issues with ECS Anywhere"
status: active
severity: MEDIUM
triggers: ["service discovery not working", "Cloud Map error", "DNS resolution failed", "service not found"]
owner: devops-agent
objective: "Resolve service discovery issues for ECS Anywhere services"
context: "Cloud Map service discovery works with ECS Anywhere but requires network connectivity between external instances and discovered endpoints."
---

## Phase 1 — Triage

MUST:
- Check service discovery configuration: `aws ecs describe-services --cluster <cluster-name> --services <service-name> --query 'services[*].serviceRegistries'`
- Check Cloud Map service instances: `aws servicediscovery list-instances --service-id <service-id>`
- Verify DNS resolution from the external instance

SHOULD:
- Check if the Cloud Map namespace is DNS-based or HTTP-based
- Verify the external instance can resolve the Cloud Map DNS namespace
- Check if health checks are configured and passing

MAY:
- Test DNS resolution from the instance: `aws ssm send-command --instance-ids <mi-id> --document-name AWS-RunShellScript --parameters commands=["dig <service-name>.<namespace>"]`
- Check Route 53 hosted zone for the namespace

## Phase 2 — Remediate

MUST:
- Ensure Cloud Map namespace DNS is resolvable from external instances
- Fix health check configuration if instances are marked unhealthy
- Verify network path exists between external instances and discovered services

SHOULD:
- Configure DNS forwarding on external instances to resolve Cloud Map namespaces
- Use HTTP-based service discovery if DNS is not feasible
- Set appropriate health check intervals and thresholds

MAY:
- Set up VPN or Direct Connect for cross-network service discovery
- Implement client-side service discovery as a fallback

## Common Issues

- symptoms: "Service registered in Cloud Map but external instances cannot resolve DNS"
  diagnosis: "External instance DNS does not include the Cloud Map private DNS namespace."
  resolution: "Configure DNS forwarding on the instance to resolve the Cloud Map namespace via Route 53 Resolver or VPN DNS."

- symptoms: "Service instances show unhealthy in Cloud Map"
  diagnosis: "Health checks fail because Cloud Map cannot reach external instances."
  resolution: "Use custom health checks or ECS-managed health checks instead of Route 53 health checks."

## Safety Ratings

```
safety_ratings:
  - "ecs describe-services: GREEN — read-only"
  - "servicediscovery list-instances: GREEN — read-only"
  - "ssm send-command: RED — executes commands on remote instances"
  - "servicediscovery update-service: YELLOW — modifies service discovery settings"
```

## Escalation Conditions

- DNS infrastructure changes required for Cloud Map resolution
- VPN or Direct Connect needed for cross-network discovery
- Health check configuration conflicts with network topology
- Service discovery latency is unacceptable for the use case

## Data Sensitivity

- `aws ecs describe-services`: MEDIUM — reveals service configuration
- `aws servicediscovery list-instances`: MEDIUM — reveals instance IPs and ports
- DNS resolution results: LOW — service endpoints
- Health check results: LOW — availability status

## Prohibited Actions

- NEVER expose Cloud Map DNS namespaces publicly
- NEVER disable health checks to work around discovery issues
- NEVER hardcode service endpoints instead of using discovery
- NEVER modify Route 53 hosted zones without understanding the impact on other services

## Phase 3 — Rollback

1. If service discovery configuration was changed, revert to previous settings
2. If DNS configuration was modified on instances, restore previous settings
3. If health check settings were changed, restore previous thresholds
4. Verify service discovery works correctly after rollback
5. Document the service discovery architecture and requirements

## Output Format

```yaml
root_cause: "service_discovery — <specific_cause>"
evidence:
  - type: discovery_error
    content: "<DNS resolution and Cloud Map details>"
severity: MEDIUM
mitigation:
  immediate: "Fix DNS or health check configuration"
  long_term: "Implement robust service discovery with fallback mechanisms"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-tasks"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-services"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "execute-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest running containers as root to fix permission issues"
  - "NEVER suggest disabling task networking"
  - "NEVER remove IAM task role to troubleshoot"
