---
title: "E1 — Access Denied Errors"
description: "Diagnose access denied errors when using RAM-shared resources"
status: active
severity: HIGH
triggers:
  - "access denied"
  - "permission denied"
  - "unauthorized"
  - "cannot access shared resource"
owner: devops-agent
objective: "Identify and resolve access denied errors for RAM-shared resources"
context: "Access denied errors with shared resources can stem from RAM permission restrictions, IAM policy denials, SCP blocks, resource-specific constraints, or the resource share not being active. Both RAM permissions and IAM policies must allow the action."
---

## Phase 1 — Triage

MUST:
- Verify resource share is ACTIVE: `aws ram get-resource-shares --resource-owner OTHER-ACCOUNTS`
- Check RAM permission on the share: `aws ram list-resource-share-permissions --resource-share-arn <share-arn>`
- Verify IAM permissions in participant account: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names <action>`
- Check if the principal is associated with the share

SHOULD:
- Check for SCPs blocking the action
- Verify the specific action is in the RAM permission
- Check CloudTrail for the denied request details
- Verify the resource is in the same region

MAY:
- Check for resource-specific access constraints
- Verify no condition keys are blocking access

## Phase 2 — Remediate

MUST:
- Ensure RAM permission includes the required action
- Add IAM permissions in the participant account
- Verify the principal is properly associated with the share

SHOULD:
- Check both RAM permission and IAM policy for the specific action
- Remove SCP restrictions if they block legitimate access
- Test with a broader permission set to isolate the issue

MAY:
- Create IAM policy templates for shared resource access
- Set up access monitoring and alerting

## Common Issues

- symptoms: "AccessDeniedException for shared resource operation"
  diagnosis: "IAM policy doesn't include the action or RAM permission is too restrictive."
  resolution: "Add IAM permissions AND verify RAM permission includes the action."

- symptoms: "Access works for some actions but not others"
  diagnosis: "RAM permission or IAM policy allows some actions but not the specific one."
  resolution: "Check both RAM permission and IAM policy for the specific denied action."

## Output Format

```yaml
root_cause: "access_denied — <specific_cause>"
evidence:
  - type: ram_permission
    content: "<RAM permission actions>"
  - type: iam_policy
    content: "<IAM policy evaluation>"
  - type: scp
    content: "<SCP evaluation if applicable>"
severity: HIGH
mitigation:
  immediate: "Fix the specific permission gap"
  long_term: "Create standardized IAM policies for shared resource access"
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
