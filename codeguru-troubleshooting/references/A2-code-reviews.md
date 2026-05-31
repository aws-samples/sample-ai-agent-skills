---
title: "A2 — Code Review Failures"
description: "Diagnose code review execution failures"
status: active
severity: MEDIUM
triggers:
  - "code review failed"
  - "review stuck in pending"
  - "review error"
owner: devops-agent
objective: "Resolve code review execution failures"
context: "Code reviews can fail due to access issues, large PRs, or service errors."
---

## Phase 1 — Triage

MUST:
- Check review status: `aws codeguru-reviewer list-code-reviews --type PullRequest --states Failed`
- Review failure reason
- Verify repository access
- Check IAM permissions

SHOULD:
- Check PR size (very large PRs may timeout)
- Verify branch references are valid

## Phase 2 — Remediate

MUST:
- Fix access issues
- Retry review if transient error
- Fix IAM permissions

SHOULD:
- Break large PRs into smaller ones
- Monitor review success rate

## Common Issues

- symptoms: "Review stuck in Pending for hours"
  diagnosis: "Service backlog or access issue."
  resolution: "Check association state. Wait or retry. Contact support if persistent."

## Output Format

```yaml
root_cause: "code_review — <specific_cause>"
evidence:
  - type: review_status
    content: "<review execution details>"
severity: MEDIUM
mitigation:
  immediate: "Fix review blockers"
  long_term: "Monitor review health"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix access issues | YELLOW |
| Retry review if transient error | GREEN |
| Fix IAM permissions | YELLOW |
| Break large PRs into smaller ones | GREEN |
| Monitor review success rate | GREEN |

## Escalation Conditions

- Fix requires modifying production IAM permissions for code review
- Fix involves persistent review failures affecting development velocity

## Data Sensitivity

- **MEDIUM**: code review results expose code quality issues and potential vulnerabilities
- Failed reviews may indicate access or configuration problems in the development pipeline

## Prohibited Actions

- NEVER suggest suppressing all security findings

## Phase 3 — Rollback

1. If IAM permissions were modified, revert to previous policy
2. If repository access was changed, restore previous access configuration
3. If review was retried, no rollback needed
4. Verify rollback by confirming code reviews execute successfully

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
