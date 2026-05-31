---
title: "A1 — Reviewer Recommendations Issues"
description: "Diagnose missing or unexpected reviewer recommendations"
status: active
severity: MEDIUM
triggers:
  - "no recommendations"
  - "recommendations not appearing"
  - "reviewer not working"
owner: devops-agent
objective: "Resolve reviewer recommendation issues"
context: "Reviewer generates recommendations on pull requests and full scans for Java and Python."
---

## Phase 1 — Triage

MUST:
- Check repository association state: `aws codeguru-reviewer describe-repository-association --association-arn <arn>`
- Verify association is ASSOCIATED
- Check code review status: `aws codeguru-reviewer list-code-reviews --type PullRequest`
- Verify language is supported (Java, Python)

SHOULD:
- Check if PR has sufficient code changes
- Verify IAM permissions for reviewer
- Check for recent successful reviews

## Phase 2 — Remediate

MUST:
- Fix repository association if not ASSOCIATED
- Ensure code is in supported language
- Verify PR triggers are configured

SHOULD:
- Trigger a full repository scan to test
- Check recommendation history

## Common Issues

- symptoms: "PR merged but no recommendations appeared"
  diagnosis: "Association not in ASSOCIATED state or language not supported."
  resolution: "Check association state. Verify code is Java or Python."

## Output Format

```yaml
root_cause: "recommendations — <specific_cause>"
evidence:
  - type: review_status
    content: "<code review details>"
severity: MEDIUM
mitigation:
  immediate: "Fix association or language"
  long_term: "Monitor review coverage"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix repository association if not ASSOCIATED | YELLOW |
| Ensure code is in supported language | GREEN |
| Verify PR triggers are configured | GREEN |
| Trigger a full repository scan to test | GREEN |
| Check recommendation history | GREEN |

## Escalation Conditions

- Fix requires modifying production repository associations
- Fix involves changing code review configurations that affect development workflow

## Data Sensitivity

- **MEDIUM**: recommendations may reference specific code patterns and potential vulnerabilities
- Review results expose code quality metrics and security posture

## Prohibited Actions

- NEVER suggest suppressing all security findings

## Phase 3 — Rollback

1. If repository association was modified, restore previous association state
2. If PR trigger configuration was changed, restore previous settings
3. If a full scan was triggered, no rollback needed (read-only operation)
4. Verify rollback by confirming `aws codeguru-reviewer describe-repository-association` shows expected state

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
