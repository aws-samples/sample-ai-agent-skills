---
title: "A3 — Full Repository Scan Issues"
description: "Diagnose full repository analysis failures"
status: active
severity: MEDIUM
triggers:
  - "full scan failed"
  - "repository analysis error"
  - "scan timeout"
owner: devops-agent
objective: "Resolve full repository scan issues"
context: "Full scans analyze entire repository. Must be explicitly triggered."
---

## Phase 1 — Triage

MUST:
- Check scan status: `aws codeguru-reviewer list-code-reviews --type RepositoryAnalysis`
- Verify repository association
- Check S3 bucket for source code (if using S3 source)
- Review scan error messages

SHOULD:
- Check repository size
- Verify branch reference
- Check IAM permissions for S3 access

## Phase 2 — Remediate

MUST:
- Fix source access issues
- Correct branch references
- Retry scan

SHOULD:
- Monitor scan costs
- Schedule scans during off-peak

## Common Issues

- symptoms: "Full scan fails with AccessDenied"
  diagnosis: "IAM role cannot access repository or S3 source."
  resolution: "Update IAM permissions. Verify S3 bucket policy."

## Output Format

```yaml
root_cause: "full_scan — <specific_cause>"
evidence:
  - type: scan_status
    content: "<scan details>"
severity: MEDIUM
mitigation:
  immediate: "Fix scan access"
  long_term: "Automate periodic scans"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix source access issues | YELLOW |
| Correct branch references | GREEN |
| Retry scan | GREEN |
| Monitor scan costs | GREEN |
| Schedule scans during off-peak | GREEN |

## Escalation Conditions

- Fix requires modifying production S3 bucket or repository access
- Fix involves persistent scan failures affecting security coverage

## Data Sensitivity

- **MEDIUM**: full scan results expose comprehensive code quality and security analysis
- S3 source configurations may reveal internal repository structures

## Prohibited Actions

- NEVER suggest suppressing all security findings

## Phase 3 — Rollback

1. If source access permissions were modified, revert to previous policy
2. If branch references were changed, restore previous branch configuration
3. If scan was triggered, no rollback needed (read-only operation)
4. Verify rollback by confirming scan configuration matches expected state

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
