---
title: "B1 — Supported Resource Issues"
description: "Diagnose issues with RAM-supported resource types"
status: active
severity: MEDIUM
triggers:
  - "resource type not supported"
  - "cannot share resource"
  - "unsupported resource"
  - "shareable resources"
owner: devops-agent
objective: "Identify supported resource types and resolve sharing compatibility issues"
context: "RAM supports specific resource types for sharing. Common types include VPC subnets, Transit Gateways, Route 53 Resolver rules, License Manager configurations, Aurora DB clusters, and more. Not all AWS resources can be shared through RAM."
---

## Phase 1 — Triage

MUST:
- List supported resource types: `aws ram list-resource-types`
- Verify the resource type is in the supported list
- Check if the specific resource can be shared: `aws ram list-resources --resource-owner SELF --resource-type <type>`
- Verify the resource exists and is in the correct state

SHOULD:
- Check regional availability of the resource type for sharing
- Verify the resource is not already shared in another share
- Check for resource-specific sharing prerequisites
- Review RAM documentation for the specific resource type

MAY:
- Check for alternative sharing mechanisms (resource policies, cross-account roles)
- Verify service-specific prerequisites for sharing

## Phase 2 — Remediate

MUST:
- Use only supported resource types for RAM sharing
- For unsupported types, use alternative sharing mechanisms (resource policies, cross-account IAM roles)
- Verify resource-specific prerequisites are met

SHOULD:
- Document which resource types are shareable in your organization
- Create guidelines for choosing between RAM and other sharing methods
- Test sharing with a single resource before bulk operations

MAY:
- Request feature support for additional resource types through AWS support
- Create automation for resource sharing workflows

## Common Issues

- symptoms: "Resource type not in list-resource-types output"
  diagnosis: "The resource type is not supported by RAM."
  resolution: "Use resource policies or cross-account IAM roles for unsupported types."

- symptoms: "Resource exists but not listed in list-resources"
  diagnosis: "Resource may not be in a shareable state or region."
  resolution: "Verify resource state and region. Some resources need specific configurations before sharing."

## Output Format

```yaml
root_cause: "supported_resources — <specific_cause>"
evidence:
  - type: resource_types
    content: "<supported resource types>"
  - type: resource_status
    content: "<resource sharing eligibility>"
severity: MEDIUM
mitigation:
  immediate: "Use supported resource type or alternative sharing method"
  long_term: "Document sharing capabilities and create resource sharing guidelines"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Resource share configurations may contain account IDs and organizational structure
- Principal ARNs reveal cross-account relationships
- Permission policies may expose internal access patterns
- Classification: INTERNAL - contains organizational topology data

## Prohibited Actions

- NEVER modify or delete resource shares in production without owner approval
- NEVER change RAM permissions that could revoke cross-account access to shared resources
- NEVER disable organization sharing without coordinating with all member accounts
- NEVER remove principals from active resource shares without impact analysis

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
