---
title: "D2 — Custom Permission Issues"
description: "Diagnose RAM custom permission creation and management problems"
status: active
severity: MEDIUM
triggers:
  - "custom permission error"
  - "create permission failed"
  - "custom permission not working"
  - "permission creation"
owner: devops-agent
objective: "Identify and resolve RAM custom permission issues"
context: "RAM custom permissions allow fine-grained access control for shared resources beyond default managed permissions. Custom permissions define specific actions participants can perform. They must be valid for the resource type and follow RAM permission syntax."
---

## Phase 1 — Triage

MUST:
- List custom permissions: `aws ram list-permissions --permission-type CUSTOMER_MANAGED`
- Get permission details: `aws ram get-permission --permission-arn <arn>`
- Verify the permission is attached to a resource share
- Check permission actions are valid for the resource type

SHOULD:
- Compare custom permission with default managed permission
- Verify permission syntax is correct
- Check CloudTrail for permission creation/update events
- Test permission with a simple operation

MAY:
- Review permission versions
- Check for conflicts with IAM policies

## Phase 2 — Remediate

MUST:
- Ensure custom permission actions are valid for the resource type
- Attach the custom permission to the resource share
- Verify participants can perform intended actions

SHOULD:
- Start with the default managed permission and customize from there
- Test custom permissions in a non-production environment
- Document custom permission rationale

MAY:
- Create permission templates for common use cases
- Automate custom permission management

## Common Issues

- symptoms: "Custom permission creation failed"
  diagnosis: "Invalid action or resource type in permission definition."
  resolution: "Verify actions are valid for the resource type. Check RAM permission syntax."

- symptoms: "Custom permission too restrictive"
  diagnosis: "Missing required actions for the participant's use case."
  resolution: "Add missing actions to the custom permission. Compare with default managed permission."

## Output Format

```yaml
root_cause: "custom_permission — <specific_cause>"
evidence:
  - type: permission_config
    content: "<custom permission definition>"
  - type: validation
    content: "<permission validation results>"
severity: MEDIUM
mitigation:
  immediate: "Fix custom permission definition"
  long_term: "Create permission templates and testing procedures"
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
