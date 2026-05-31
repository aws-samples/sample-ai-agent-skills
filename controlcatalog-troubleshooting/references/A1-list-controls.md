---
title: "A1 — List Controls"
description: "Diagnose control listing issues"
status: active
severity: MEDIUM
triggers: ["cannot list controls", "empty control list", "list-controls error"]
owner: devops-agent
objective: "Resolve control listing issues"
context: "Control listing returns available controls. Empty results may indicate permissions, pagination, or regional issues."
---

## Phase 1 — Triage

MUST:
- Try listing controls: `aws controlcatalog list-controls`
- Check IAM permissions for controlcatalog:ListControls
- Verify API endpoint is available in the region

SHOULD:
- Check for pagination (use --max-results and --next-token)
- Verify AWS CLI version supports controlcatalog commands

MAY:
- Check AWS Health Dashboard for service issues

## Phase 2 — Remediate

MUST:
- Add controlcatalog:ListControls to IAM policy
- Handle pagination for complete results

SHOULD:
- Update AWS CLI to latest version

MAY:
- Use SDK for programmatic access with pagination handling

## Common Issues

- symptoms: "list-controls returns empty results"
  diagnosis: "IAM permissions missing or CLI version too old."
  resolution: "Add controlcatalog:* permissions. Update AWS CLI."

## Output Format

```yaml
root_cause: "list_controls — <specific_cause>"
evidence:
  - type: api_response
    content: "<list-controls response or error>"
severity: MEDIUM
mitigation:
  immediate: "Fix permissions or CLI version"
  long_term: "Automate control discovery"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Control catalog data is public AWS reference information |
| Blast Radius | LOW — Read-only API; no infrastructure changes |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | MEDIUM — Control listing supports compliance programs |

## Escalation Conditions

- Escalate to security team if control listing failures block compliance assessments
- Escalate to AWS Support if API consistently returns empty results despite correct permissions

## Data Sensitivity

- **Control catalog data** is AWS-published reference information — low sensitivity
- **IAM permissions** for controlcatalog access should follow least-privilege

## PROHIBITED Actions

- Do NOT assume control catalog is a complete compliance solution — it is a reference tool
- Do NOT modify IAM permissions broadly to fix listing issues — scope to controlcatalog:* only

## Phase 3 — Rollback

- No rollback needed — read-only API
- If IAM permissions were changed during troubleshooting, revert to previous policy

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
