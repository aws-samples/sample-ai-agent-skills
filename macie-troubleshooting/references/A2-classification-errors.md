---
title: "A2 — Classification Errors"
description: "Diagnose classification accuracy issues in Macie"
status: active
severity: MEDIUM
triggers:
  - "wrong classification"
  - "missed sensitive data"
  - "incorrect detection"
  - "classification inaccurate"
owner: devops-agent
objective: "Improve Macie classification accuracy for sensitive data detection"
context: "Classification errors occur when Macie misidentifies data (false positives) or misses sensitive data (false negatives). Causes include unsupported file formats, disabled managed identifiers, missing custom identifiers, or allow list misconfiguration."
---

## Phase 1 — Triage

MUST:
- Check which managed identifiers are enabled for the job: `aws macie2 describe-classification-job --job-id <job-id>`
- Check custom data identifiers: `aws macie2 list-custom-data-identifiers`
- Check allow lists: `aws macie2 list-allow-lists`
- Review finding details for classification results: `aws macie2 get-findings --finding-ids <finding-id>`

SHOULD:
- Verify the file format is supported by Macie
- Check if the sensitive data pattern matches managed identifier definitions
- Review custom identifier regex patterns for accuracy

MAY:
- Test custom identifiers with sample data: `aws macie2 test-custom-data-identifier --regex <pattern> --sample-text <text>`
- Check if allow lists are incorrectly filtering valid findings

## Phase 2 — Remediate

MUST:
- Enable relevant managed identifiers if disabled
- Create custom identifiers for organization-specific sensitive data
- Fix allow lists that incorrectly suppress valid detections

SHOULD:
- Test custom identifier patterns before deploying to production jobs
- Review and update allow lists regularly
- Use keywords with custom identifiers to improve accuracy

MAY:
- Create separate jobs with different identifier configurations for testing
- Report classification issues to AWS if managed identifiers are inaccurate

## Common Issues

- symptoms: "Known sensitive data not detected"
  diagnosis: "File format unsupported or managed identifier disabled."
  resolution: "Check file format support. Enable relevant managed identifiers."

- symptoms: "Too many false positives"
  diagnosis: "Managed identifiers matching non-sensitive patterns."
  resolution: "Create allow lists for known safe values. Use suppression for accepted patterns."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable managed identifiers | GREEN | Additive detection capability |
| Create custom identifiers | GREEN | New resource creation |
| Fix allow lists | YELLOW | May expose previously suppressed findings |
| Test custom identifiers | GREEN | Read-only test operation |

## Escalation Conditions

- Known sensitive data not being detected — escalate to security team
- High false positive rate affecting triage — escalate to security operations
- Custom identifier changes for compliance requirements — escalate to compliance team
- Allow list modifications — escalate to data classification team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws macie2 describe-classification-job` | HIGH | Exposes job configuration and identifiers |
| `aws macie2 get-findings` | HIGH | Exposes sensitive data detection results |
| `aws macie2 list-custom-data-identifiers` | MEDIUM | Exposes custom detection patterns |
| `aws macie2 list-allow-lists` | MEDIUM | Exposes allowed data patterns |

## Prohibited Actions

- **NEVER** suggest disabling Macie as a troubleshooting step
- **NEVER** suggest ignoring sensitive data findings without review
- **NEVER** disable all managed identifiers without understanding the detection impact
- **NEVER** create overly broad allow lists that suppress valid sensitive data findings

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled managed identifiers | Disable specific identifiers in job configuration |
| Created custom identifiers | Delete: `aws macie2 delete-custom-data-identifier --id <id>` |
| Modified allow lists | Restore previous allow list configuration |

## Output Format

```yaml
root_cause: "classification — <specific_cause>"
evidence:
  - type: identifiers
    content: "<enabled managed and custom identifiers>"
  - type: findings
    content: "<classification results>"
severity: MEDIUM
mitigation:
  immediate: "Adjust identifiers and allow lists"
  long_term: "Establish regular classification accuracy review process"
```

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
  - "NEVER suggest disabling Macie to reduce costs without review"
  - "NEVER suggest ignoring sensitive data findings"
  - "NEVER suggest removing S3 bucket monitoring"
