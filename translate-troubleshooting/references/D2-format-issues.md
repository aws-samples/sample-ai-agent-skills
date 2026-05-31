---
title: "D2 — Document Format Issues"
description: "Diagnose document format problems in batch translation"
status: active
severity: MEDIUM
triggers:
  - "format not supported"
  - "HTML translation error"
  - "DOCX translation failed"
owner: devops-agent
objective: "Identify and resolve document format issues"
context: "Batch translation supports TXT, HTML, DOCX, XLIFF. HTML preserves tags. DOCX preserves formatting. ContentType must match actual format."
---

## Phase 1 — Triage

MUST:
- Verify file format matches ContentType in job configuration
- Check file encoding (UTF-8 recommended)
- Test with a single file of the same format
- Review per-file errors in job output

SHOULD:
- Validate HTML structure if using HTML format
- Check DOCX file integrity
- Verify XLIFF version compatibility

MAY:
- Convert problematic files to TXT as fallback
- Check for special characters causing issues

## Phase 2 — Remediate

MUST:
- Match ContentType to actual file format
- Use UTF-8 encoding
- Fix malformed HTML/DOCX files

SHOULD:
- Validate files before batch submission
- Use TXT for simple text translation

MAY:
- Implement format validation pipeline

## Common Issues

- symptoms: "HTML tags appear in translated text"
  diagnosis: "ContentType not set to text/html."
  resolution: "Set ContentType to text/html for HTML files."

- symptoms: "DOCX translation loses formatting"
  diagnosis: "Complex DOCX features not fully supported."
  resolution: "Simplify DOCX formatting. Test with sample file."

## Output Format

```yaml
root_cause: "format_issues — <specific_cause>"
evidence:
  - type: file_format
    content: "<file format details>"
severity: MEDIUM
mitigation:
  immediate: "Fix ContentType or file format"
  long_term: "Implement format validation before batch submission"
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

- Source documents may contain business-sensitive content
- Custom terminologies reveal domain-specific vocabulary
- Parallel data contains paired translation examples
- Classification: CONFIDENTIAL - business content and proprietary terminology

## Prohibited Actions

- NEVER delete custom terminologies used by production translation workflows
- NEVER modify parallel data sets during active batch translation jobs
- NEVER change formality settings on production translation endpoints without review
- NEVER process documents containing PII without confirming data handling compliance

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
