---
title: "B1 — Finding Analysis"
description: "Analyze and prioritize Macie sensitive data findings"
status: active
severity: MEDIUM
triggers:
  - "too many findings"
  - "prioritize findings"
  - "sensitive data found"
  - "finding details"
owner: devops-agent
objective: "Analyze Macie findings to prioritize remediation of sensitive data exposure"
context: "Macie findings indicate sensitive data detected in S3 objects. Prioritization should consider data type sensitivity, bucket public access, encryption status, and business criticality. Findings include location details for precise remediation."
---

## Phase 1 — Triage

MUST:
- Get finding statistics: `aws macie2 get-finding-statistics --finding-criteria '{}' --group-by '{"key":"severity.description"}'`
- List high-severity findings: `aws macie2 list-findings --finding-criteria '{"criterion":{"severity.description":{"eq":["High"]}}}' --max-results 20`
- Get finding details: `aws macie2 get-findings --finding-ids <finding-id>`
- Check sensitive data occurrences: `aws macie2 get-sensitive-data-occurrences --finding-id <finding-id>`

SHOULD:
- Group findings by bucket: `aws macie2 get-finding-statistics --finding-criteria '{}' --group-by '{"key":"resourcesAffected.s3Bucket.name"}'`
- Check if affected buckets are publicly accessible
- Review finding confidence scores

MAY:
- Export findings for detailed analysis
- Correlate with S3 access logs for exposure assessment

## Phase 2 — Remediate

MUST:
- Address high-severity findings in publicly accessible buckets first
- Review sensitive data locations and determine appropriate action
- Encrypt unencrypted buckets containing sensitive data

SHOULD:
- Restrict bucket access using bucket policies
- Enable S3 Block Public Access on affected buckets
- Create remediation tickets for each high-severity finding

MAY:
- Set up automated remediation with EventBridge + Lambda
- Create dashboards for finding trends

## Common Issues

- symptoms: "Overwhelmed by finding volume"
  diagnosis: "Need to prioritize by severity and exposure."
  resolution: "Filter by High severity + public buckets. Address those first."

- symptoms: "Same data type across many buckets"
  diagnosis: "Systemic sensitive data storage issue."
  resolution: "Address at organizational level with data governance policies."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Address high-severity findings | GREEN | Remediation of data exposure |
| Encrypt unencrypted buckets | YELLOW | Encryption changes affect all bucket access |
| Restrict bucket access | YELLOW | Policy changes affect all bucket users |
| Enable S3 Block Public Access | GREEN | Security enhancement |

## Escalation Conditions

- High-severity findings in publicly accessible buckets — escalate immediately to security team
- Sensitive data found in non-compliant storage — escalate to compliance team
- Large-scale remediation across many buckets — escalate to infrastructure team
- Findings indicating data breach — escalate to incident response team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws macie2 get-findings` | HIGH | Exposes sensitive data detection details and locations |
| `aws macie2 get-sensitive-data-occurrences` | HIGH | Exposes actual sensitive data samples |
| `aws macie2 get-finding-statistics` | MEDIUM | Exposes finding counts by severity |

## Prohibited Actions

- **NEVER** suggest disabling Macie as a troubleshooting step
- **NEVER** suggest ignoring sensitive data findings without review
- **NEVER** share sensitive data occurrence details without proper authorization
- **NEVER** access sensitive data samples without a documented business need

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled bucket encryption | Cannot easily revert (existing objects remain encrypted) |
| Restricted bucket access | Restore previous bucket policy |
| Enabled Block Public Access | Disable specific Block Public Access settings if needed |

## Output Format

```yaml
root_cause: "finding_analysis — <specific_focus>"
evidence:
  - type: severity_summary
    content: "<finding counts by severity>"
  - type: high_findings
    content: "<high-severity finding details>"
severity: MEDIUM
mitigation:
  immediate: "Address high-severity findings in public buckets"
  long_term: "Establish data governance and regular scanning"
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
