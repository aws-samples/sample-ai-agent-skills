---
title: "D1 — Managed Permission Issues"
description: "Diagnose RAM managed permission problems"
status: active
severity: MEDIUM
triggers:
  - "managed permission error"
  - "default permission"
  - "permission not working"
  - "RAM permission denied"
owner: devops-agent
objective: "Identify and resolve RAM managed permission issues"
context: "RAM managed permissions define what actions participants can perform on shared resources. Each resource type has a default managed permission. Managed permissions can have multiple versions. Permission updates require explicit version selection."
---

## Phase 1 — Triage

MUST:
- List available permissions: `aws ram list-permissions --resource-type <type>`
- Get permission details: `aws ram get-permission --permission-arn <arn>`
- Check which permission is attached to the share: `aws ram list-resource-share-permissions --resource-share-arn <share-arn>`
- Verify the permission version in use

SHOULD:
- Check for newer permission versions: `aws ram list-permission-versions --permission-arn <arn>`
- Verify the permission actions match the required access
- Check if a custom permission would be more appropriate
- Review IAM policies in participant account for complementary permissions

MAY:
- Compare default vs available managed permissions
- Check for permission deprecation notices

## Phase 2 — Remediate

MUST:
- Ensure the correct managed permission is attached to the share
- Verify permission actions cover the required operations
- Update permission version if a newer version is needed

SHOULD:
- Use the most restrictive permission that meets requirements
- Document which permissions are used for each resource type
- Test permission changes with a single participant first

MAY:
- Create custom permissions for fine-grained access
- Set up permission version monitoring

## Common Issues

- symptoms: "Participant can't perform expected action on shared resource"
  diagnosis: "Managed permission doesn't include the required action."
  resolution: "Check permission actions. Use a different managed permission or create a custom one."

- symptoms: "Permission version outdated"
  diagnosis: "Newer permission version available with additional actions."
  resolution: "Update to the latest permission version: aws ram list-permission-versions."

## Output Format

```yaml
root_cause: "managed_permission — <specific_cause>"
evidence:
  - type: permission_config
    content: "<managed permission details and version>"
  - type: share_permissions
    content: "<permissions attached to the share>"
severity: MEDIUM
mitigation:
  immediate: "Fix permission configuration"
  long_term: "Implement permission version management and monitoring"
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
