---
title: "D2 — Milestone Comparison Issues"
description: "Diagnose milestone comparison and progress tracking"
status: active
severity: LOW
triggers: ["compare milestones", "progress tracking", "milestone diff"]
owner: devops-agent
objective: "Resolve milestone comparison issues"
context: "Milestones enable tracking improvement progress over time."
---

## Phase 1 — Triage

MUST:
- List milestones: `aws wellarchitected list-milestones --workload-id <id>`
- Get milestone details: `aws wellarchitected get-milestone --workload-id <id> --milestone-number <num>`
- Compare risk counts between milestones

## Phase 2 — Remediate

MUST:
- Create milestones at regular intervals
- Compare risk counts to track progress

## Common Issues

- symptoms: "Cannot see improvement between milestones"
  diagnosis: "Answers not updated between milestones."
  resolution: "Update answers before creating new milestone."

## Output Format

```yaml
root_cause: "milestone_comparison — <specific_cause>"
evidence:
  - type: milestone_diff
    content: "<milestone comparison>"
severity: LOW
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

- Workload reviews contain detailed architecture assessments
- Risk findings reveal infrastructure weaknesses
- Custom lenses may contain proprietary review criteria
- Classification: CONFIDENTIAL - architecture assessments and risk findings

## Prohibited Actions

- NEVER delete workload reviews containing compliance-required assessments
- NEVER modify custom lenses published to production accounts
- NEVER dismiss high-risk findings without documented justification and approval
- NEVER remove shared workload access without notifying collaborators

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
