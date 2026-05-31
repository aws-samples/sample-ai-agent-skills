---
title: "E2 — Delegate Access"
description: "Diagnose delegate access issues"
status: active
severity: MEDIUM
triggers: ["delegate cannot access", "permission denied for delegate"]
owner: devops-agent
objective: "Resolve delegate access issues"
context: "Delegates need IAM permissions to access their assigned control sets. Delegation alone doesn't grant IAM access."
---

## Phase 1 — Triage

MUST:
- Verify delegate's IAM permissions include auditmanager actions
- Check if delegation is active and not expired
- Verify delegate is accessing the correct assessment

SHOULD:
- Check for SCPs blocking Audit Manager access
- Verify delegate's IAM role/user is correct

MAY:
- Test access with delegate credentials

## Phase 2 — Remediate

MUST:
- Add required Audit Manager permissions to delegate's IAM policy
- Verify delegation is active

SHOULD:
- Use IAM policy with resource-level scoping to the specific assessment

MAY:
- Create dedicated IAM role for Audit Manager delegates

## Common Issues

- symptoms: "Delegate gets AccessDeniedException"
  diagnosis: "IAM policy missing auditmanager permissions."
  resolution: "Add auditmanager:Get*, auditmanager:List*, auditmanager:UpdateAssessmentControl to delegate IAM."

## Output Format

```yaml
root_cause: "delegate_access — <specific_cause>"
evidence:
  - type: access_error
    content: "<IAM permission details>"
severity: MEDIUM
mitigation:
  immediate: "Fix IAM permissions"
  long_term: "Create standard delegate IAM role"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Add required Audit Manager permissions to delegate IAM | YELLOW | IAM permission change — review scope |
| Verify delegation is active | GREEN | Verification only |
| Use IAM policy with resource-level scoping | GREEN | Best practice — more restrictive |
| Create dedicated IAM role for delegates | YELLOW | New IAM role — review permissions carefully |

## Escalation Conditions

- Fix requires modifying compliance framework used by auditors
- Assessment covers production accounts with active compliance requirements
- IAM permission changes affect compliance data access
- Delegate access issues may delay compliance review timelines

## Data Sensitivity

- Delegate access controls visibility into compliance evidence and findings
- IAM policies reveal the scope of compliance data accessible to delegates
- Access errors may indicate unauthorized access attempts

## Prohibited Actions

- NEVER suggest deleting assessments to fix compliance gaps — this destroys audit evidence
- NEVER grant wildcard Audit Manager permissions to delegates
- NEVER share delegate credentials or access keys

## Phase 3 — Rollback

- If IAM permissions were granted incorrectly: revert delegate IAM policy to previous version
- If dedicated role was created incorrectly: delete the role after removing all assignments
- Document all IAM changes for compliance audit trail

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
