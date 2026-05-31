---
title: "F2 — Report Access"
description: "Diagnose report access and download issues"
status: active
severity: MEDIUM
triggers: ["cannot download report", "report URL expired", "report access denied"]
owner: devops-agent
objective: "Resolve report access issues"
context: "Reports are stored in S3 with presigned URLs for download. URLs expire after a period."
---

## Phase 1 — Triage

MUST:
- Get report URL: `aws auditmanager get-assessment-report-url --assessment-id <id> --assessment-report-id <report-id>`
- Check if URL has expired
- Verify S3 bucket access permissions

SHOULD:
- Check if report was generated successfully
- Verify S3 object exists

MAY:
- Check S3 bucket versioning for report history

## Phase 2 — Remediate

MUST:
- Regenerate report URL if expired
- Fix S3 access permissions if denied

SHOULD:
- Download reports promptly after generation

MAY:
- Set up automated report distribution

## Common Issues

- symptoms: "Report URL returns 403 Forbidden"
  diagnosis: "Presigned URL expired or S3 permissions changed."
  resolution: "Generate new report URL: aws auditmanager get-assessment-report-url."

## Output Format

```yaml
root_cause: "report_access — <specific_cause>"
evidence:
  - type: access_details
    content: "<report URL and access status>"
severity: MEDIUM
mitigation:
  immediate: "Regenerate report URL"
  long_term: "Automate report distribution"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Regenerate report URL if expired | GREEN | Creates new presigned URL — non-destructive |
| Fix S3 access permissions if denied | YELLOW | Permission change — affects report access |
| Download reports promptly | GREEN | Operational guidance only |
| Set up automated report distribution | YELLOW | New automation — review distribution list |

## Escalation Conditions

- Fix requires modifying compliance framework used by auditors
- Assessment covers production accounts with active compliance requirements
- Report access changes affect who can view compliance findings
- Automated distribution may send sensitive reports to unintended recipients

## Data Sensitivity

- Reports contain sensitive compliance findings and evidence
- Presigned URLs provide temporary access to compliance documentation
- Report distribution must be limited to authorized compliance reviewers

## Prohibited Actions

- NEVER suggest deleting assessments to fix compliance gaps — this destroys audit evidence
- NEVER share report URLs or reports with unauthorized parties
- NEVER distribute compliance reports via unencrypted channels

## Phase 3 — Rollback

- If S3 permissions were changed incorrectly: restore original access policy
- If automated distribution was configured incorrectly: disable the distribution mechanism
- Document all report access changes for compliance audit trail

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
