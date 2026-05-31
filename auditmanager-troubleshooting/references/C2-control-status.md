---
title: "C2 — Control Status"
description: "Diagnose control status interpretation issues"
status: active
severity: MEDIUM
triggers: ["control status unclear", "under review", "reviewed status"]
owner: devops-agent
objective: "Resolve control status issues"
context: "Controls have statuses: Under review, Reviewed, Inactive. Status is set by reviewers, not automatically."
---

## Phase 1 — Triage

MUST:
- Check control status in assessment
- Verify who last reviewed the control
- Check evidence completeness for the control

SHOULD:
- Review evidence quality and relevance
- Check if manual evidence is needed

MAY:
- Compare control status across assessments

## Phase 2 — Remediate

MUST:
- Review evidence and update control status appropriately
- Add comments explaining status decisions

SHOULD:
- Assign controls to appropriate reviewers
- Set review deadlines

MAY:
- Automate status updates based on evidence completeness

## Common Issues

- symptoms: "All controls stuck in Under Review"
  diagnosis: "No reviewer has marked controls as Reviewed."
  resolution: "Assign reviewers and set review schedule."

## Output Format

```yaml
root_cause: "control_status — <specific_cause>"
evidence:
  - type: status_details
    content: "<control status and review history>"
severity: MEDIUM
mitigation:
  immediate: "Update control statuses"
  long_term: "Implement review workflow with deadlines"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Review evidence and update control status | YELLOW | Changes compliance status — requires auditor judgment |
| Add comments explaining status decisions | GREEN | Documentation only — non-destructive |
| Assign controls to appropriate reviewers | GREEN | Delegation — non-destructive |
| Automate status updates based on evidence | YELLOW | Automation may bypass manual review requirements |

## Escalation Conditions

- Fix requires modifying compliance framework used by auditors
- Assessment covers production accounts with active compliance requirements
- Control status changes affect overall compliance posture reporting
- Status updates during active audit periods require auditor approval

## Data Sensitivity

- Control status reveals compliance posture for specific requirements
- Review comments may contain sensitive compliance findings
- Status history reveals compliance timeline and gaps

## Prohibited Actions

- NEVER suggest deleting assessments to fix compliance gaps — this destroys audit evidence
- NEVER mark controls as "Reviewed" without actual evidence review
- NEVER automate status changes for controls requiring manual attestation

## Phase 3 — Rollback

- If control status was changed incorrectly: update status back to "Under review" with explanation comment
- If reviewer assignments were wrong: reassign to correct reviewers
- Document all status changes with justification for audit trail

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
