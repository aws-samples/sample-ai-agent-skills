---
title: "A2 — App Mesh Configuration Issues"
description: "Diagnose mesh-level configuration problems"
status: active
severity: MEDIUM
triggers:
  - "mesh configuration"
  - "egress filter"
  - "mesh settings"
  - "mesh policy"
owner: devops-agent
objective: "Identify and resolve mesh-level configuration issues affecting service communication"
context: "Mesh configuration includes egress filter policy, service discovery type, and IP preference. Misconfiguration affects how services communicate within and outside the mesh."
---

## Phase 1 — Triage

MUST:
- Check mesh configuration: `aws appmesh describe-mesh --mesh-name <mesh-name>`
- Check egress filter: `aws appmesh describe-mesh --mesh-name <mesh-name> --query 'mesh.spec.egressFilter'`
- List all virtual nodes: `aws appmesh list-virtual-nodes --mesh-name <mesh-name>`
- List all virtual services: `aws appmesh list-virtual-services --mesh-name <mesh-name>`

SHOULD:
- Check if IP preference is configured (IPv4, IPv6, or dual-stack)
- Verify service discovery type matches infrastructure setup
- Review mesh metadata and tags

MAY:
- Check cross-account sharing configuration via RAM
- Review mesh resource quotas

## Phase 2 — Remediate

MUST:
- Set egress filter appropriately: ALLOW_ALL for permissive, DROP_ALL for strict
- Ensure all required services are registered in the mesh when using DROP_ALL
- Configure IP preference to match VPC and service configuration

SHOULD:
- Document egress filter policy and rationale
- Plan migration from ALLOW_ALL to DROP_ALL incrementally
- Verify all virtual nodes have correct backends configured

MAY:
- Implement mesh configuration as code for consistency
- Set up monitoring for mesh configuration changes

## Common Issues

- symptoms: "Services cannot reach external endpoints"
  diagnosis: "Egress filter set to DROP_ALL blocking traffic to non-mesh destinations."
  resolution: "Set egress filter to ALLOW_ALL or add virtual services for external endpoints."

- symptoms: "IPv6 services unreachable"
  diagnosis: "Mesh IP preference not configured for IPv6 or dual-stack."
  resolution: "Update mesh IP preference to match service addressing."

- symptoms: "Cross-account services cannot join mesh"
  diagnosis: "Mesh not shared via RAM to the target account."
  resolution: "Share the mesh using AWS RAM with the target account."

## Output Format

```yaml
root_cause: "mesh_configuration_issue — <specific_cause>"
evidence:
  - type: mesh_config
    content: "<egress filter, IP preference, service discovery>"
  - type: virtual_resources
    content: "<virtual nodes and services in mesh>"
severity: MEDIUM
mitigation:
  immediate: "Update mesh configuration to fix the identified issue"
  long_term: "Implement mesh configuration management and monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Set egress filter to ALLOW_ALL | YELLOW | Changes traffic routing policy |
| Set egress filter to DROP_ALL | RED | May break external connectivity |
| Configure IP preference | YELLOW | Affects service addressing |
| Document egress filter policy | GREEN | Documentation only |
| Implement mesh config as code | GREEN | IaC setup |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Mesh configuration**: Egress filter, IP preference — INTERNAL
- **Virtual resources**: Node and service topology — INTERNAL
- **Cross-account config**: RAM sharing details — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER switch egress filter from ALLOW_ALL to DROP_ALL without verifying all services are registered
- NEVER modify mesh configuration during active traffic incidents

## Phase 3 — Rollback

1. Revert egress filter to previous setting: `aws appmesh update-mesh --mesh-name <mesh-name> --spec egressFilter={type=ALLOW_ALL}`
2. Revert IP preference to previous configuration
3. Verify all virtual nodes and services are still communicating
4. Check Envoy proxy health across all pods/tasks
5. Monitor error rates for 15 minutes after rollback

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
