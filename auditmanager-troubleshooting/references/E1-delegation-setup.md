---
title: "E1 — Delegation Setup"
description: "Diagnose delegation setup issues"
status: active
severity: MEDIUM
triggers: ["cannot delegate", "delegation failed", "assign control set"]
owner: devops-agent
objective: "Resolve delegation setup issues"
context: "Delegation assigns control sets to team members for review. Requires valid IAM principals and permissions."
---

## Phase 1 — Triage

MUST:
- Verify delegate IAM principal exists
- Check delegation permissions for the assessment owner
- Verify control set exists in the assessment

SHOULD:
- Check if delegate has Audit Manager IAM permissions
- Verify delegation notification settings

MAY:
- Review existing delegations

## Phase 2 — Remediate

MUST:
- Ensure delegate IAM principal is valid
- Grant delegate auditmanager permissions

SHOULD:
- Notify delegates of their assignments
- Set review deadlines

MAY:
- Automate delegation for recurring assessments

## Common Issues

- symptoms: "Delegation created but delegate cannot access"
  diagnosis: "Delegate lacks IAM permissions for Audit Manager."
  resolution: "Add auditmanager:Get*, auditmanager:List*, auditmanager:Update* to delegate's IAM policy."

## Output Format

```yaml
root_cause: "delegation_setup — <specific_cause>"
evidence:
  - type: delegation_config
    content: "<delegation status and permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix delegation permissions"
  long_term: "Standardize delegation process"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure delegate IAM principal is valid | GREEN | Verification only |
| Grant delegate auditmanager permissions | YELLOW | IAM permission change — review scope |
| Notify delegates of assignments | GREEN | Communication only |
| Automate delegation for recurring assessments | YELLOW | New automation — test before production |

## Escalation Conditions

- Fix requires modifying compliance framework used by auditors
- Assessment covers production accounts with active compliance requirements
- Delegation changes affect who can review and modify compliance controls
- IAM permission grants for delegates must follow least-privilege

## Data Sensitivity

- Delegation configuration reveals organizational compliance review structure
- Delegate access grants visibility into compliance posture and evidence
- Assessment delegation may expose sensitive compliance findings to new reviewers

## Prohibited Actions

- NEVER suggest deleting assessments to fix compliance gaps — this destroys audit evidence
- NEVER grant delegates more permissions than needed for their assigned control sets
- NEVER delegate controls to principals outside the compliance review team

## Phase 3 — Rollback

- If delegation was created incorrectly: remove the delegation from the assessment
- If IAM permissions were granted incorrectly: revert delegate IAM policy
- Document all delegation changes for compliance audit trail

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
