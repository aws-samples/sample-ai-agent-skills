---
title: "B2 — False Positives"
description: "Manage false positive findings in Macie"
status: active
severity: LOW
triggers:
  - "false positive"
  - "incorrect finding"
  - "not sensitive data"
  - "wrong detection"
owner: devops-agent
objective: "Identify and manage false positive Macie findings"
context: "False positives occur when Macie flags non-sensitive data as sensitive. Common causes include test data matching PII patterns, public data matching identifier patterns, or overly broad custom identifiers. Allow lists and suppression rules help manage false positives."
---

## Phase 1 — Triage

MUST:
- Get finding details: `aws macie2 get-findings --finding-ids <finding-id>`
- Check sensitive data occurrences: `aws macie2 get-sensitive-data-occurrences --finding-id <finding-id>`
- Verify the flagged data is actually non-sensitive
- Check existing allow lists: `aws macie2 list-allow-lists`

SHOULD:
- Review the data identifier that triggered the finding
- Check confidence score (low confidence = higher false positive risk)
- Determine if the pattern is recurring across multiple objects

MAY:
- Check if custom identifiers are too broad
- Review managed identifier behavior for the data type

## Phase 2 — Remediate

MUST:
- Verify the finding is truly a false positive before suppressing
- Create allow list for known safe patterns: `aws macie2 create-allow-list --name <name> --criteria '{"regex":"<pattern>"}'`
- Or create suppression rule: `aws macie2 create-findings-filter --name <name> --action ARCHIVE --finding-criteria '<criteria>'`

SHOULD:
- Use allow lists for systematic false positives (test data patterns)
- Use suppression rules for specific one-off false positives
- Document false positive decisions

MAY:
- Refine custom identifiers to reduce false positives
- Report managed identifier false positives to AWS

## Common Issues

- symptoms: "Test data flagged as PII"
  diagnosis: "Test SSNs/credit cards match managed identifier patterns."
  resolution: "Create allow list with test data patterns."

- symptoms: "Public information flagged as sensitive"
  diagnosis: "Public phone numbers or addresses matching PII patterns."
  resolution: "Add to allow list or create suppression rule."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create allow list | YELLOW | Suppresses future detections matching the pattern |
| Create suppression rule | YELLOW | Hides findings from active view |
| Refine custom identifiers | GREEN | Improves detection accuracy |

## Escalation Conditions

- Uncertainty about whether data is truly non-sensitive — escalate to data classification team
- High false positive rate affecting security triage — escalate to security operations
- Allow list changes for compliance-monitored data types — escalate to compliance team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws macie2 get-findings` | HIGH | Exposes sensitive data detection details |
| `aws macie2 get-sensitive-data-occurrences` | HIGH | Exposes actual data samples |
| `aws macie2 list-allow-lists` | MEDIUM | Exposes allowed patterns |

## Prohibited Actions

- **NEVER** suggest disabling Macie as a troubleshooting step
- **NEVER** suggest ignoring sensitive data findings without review
- **NEVER** create allow lists without verifying the data is truly non-sensitive
- **NEVER** suppress findings for data types required by compliance

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created allow list | Delete: `aws macie2 delete-allow-list --id <id>` |
| Created suppression rule | Delete: `aws macie2 delete-findings-filter --id <id>` |
| Modified custom identifiers | Restore previous identifier configuration |

## Output Format

```yaml
root_cause: "false_positive — <specific_cause>"
evidence:
  - type: finding_details
    content: "<finding and data identifier>"
  - type: verification
    content: "<verification that data is not sensitive>"
severity: LOW
mitigation:
  immediate: "Create allow list or suppression rule"
  long_term: "Maintain allow lists and review false positive rates"
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
