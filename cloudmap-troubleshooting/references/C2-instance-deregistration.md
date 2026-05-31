---
title: "C2 — Cloud Map Instance Deregistration Issues"
description: "Diagnose instance deregistration and cleanup problems"
status: active
severity: MEDIUM
triggers:
  - "instance not deregistering"
  - "stale instance"
  - "orphaned instance"
  - "deregister failed"
owner: devops-agent
objective: "Identify and resolve instance deregistration issues in Cloud Map"
context: "Instance deregistration removes endpoints from a service and deletes associated DNS records. Stale instances cause traffic to be routed to non-existent endpoints. Deregistration failures leave orphaned records."
---

## Phase 1 — Triage

MUST:
- List instances: `aws servicediscovery list-instances --service-id <service-id>`
- Check instance details: `aws servicediscovery get-instance --service-id <service-id> --instance-id <instance-id>`
- Check operation status: `aws servicediscovery get-operation --operation-id <operation-id>`
- Verify DNS records still exist: `aws route53 list-resource-record-sets --hosted-zone-id <zone-id>`

SHOULD:
- Check if instance is ECS-managed (should not be manually deregistered)
- Verify IAM permissions for deregistration
- Check health status of stale instances

MAY:
- Check CloudTrail for deregistration attempts
- Verify no automation is re-registering the instance

## Phase 2 — Remediate

MUST:
- Deregister stale instances: `aws servicediscovery deregister-instance --service-id <service-id> --instance-id <instance-id>`
- Verify DNS records are removed after deregistration
- Do NOT manually deregister ECS-managed instances

SHOULD:
- Implement automated cleanup for terminated instances
- Use health checks to mark unhealthy instances
- Monitor for orphaned instances regularly

MAY:
- Set up Lambda-based cleanup for stale instances
- Implement instance lifecycle hooks

## Common Issues

- symptoms: "Instance deregistered but DNS still resolves to old IP"
  diagnosis: "DNS TTL hasn't expired yet — clients caching old records."
  resolution: "Wait for TTL expiration. Consider lowering TTL for faster updates."

- symptoms: "Cannot deregister instance — operation fails"
  diagnosis: "IAM permissions missing or instance managed by ECS."
  resolution: "Check permissions. For ECS instances, stop the ECS task instead."

- symptoms: "Stale instances accumulating in service"
  diagnosis: "No automated cleanup when compute resources are terminated."
  resolution: "Implement lifecycle hooks or Lambda-based cleanup."

## Output Format

```yaml
root_cause: "instance_deregistration_issue — <specific_cause>"
evidence:
  - type: instances
    content: "<stale instance details>"
  - type: dns_records
    content: "<orphaned DNS records>"
  - type: operation
    content: "<deregistration operation status>"
severity: MEDIUM
mitigation:
  immediate: "Deregister stale instances and verify DNS cleanup"
  long_term: "Implement automated instance lifecycle management"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Deregister stale instances | YELLOW |
| Verify DNS records are removed | GREEN |
| Do NOT manually deregister ECS-managed instances | GREEN |
| Implement automated cleanup | GREEN |
| Use health checks to mark unhealthy instances | GREEN |
| Set up Lambda-based cleanup | GREEN |

## Escalation Conditions

- Fix requires deregistering instances from production services
- Fix involves modifying automated cleanup that affects production workloads

## Data Sensitivity

- **MEDIUM**: deregistration operations affect service availability and traffic routing
- Stale instance data may reveal previous infrastructure configurations

## Prohibited Actions

- NEVER suggest deleting namespaces with active service registrations

## Phase 3 — Rollback

1. If instances were deregistered incorrectly, re-register them with correct attributes
2. If automated cleanup was implemented, disable it if it removes valid instances
3. If DNS records were manually cleaned up, verify they are recreated on re-registration
4. Verify rollback by confirming instance list and DNS records match expected state

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
