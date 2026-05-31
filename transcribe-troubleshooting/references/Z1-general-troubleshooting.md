---
title: "Z1 — General Transcribe Troubleshooting"
description: "Catch-all runbook for Transcribe issues"
status: active
severity: MEDIUM
triggers: ["transcribe error", "transcribe not working", "transcription issue"]
owner: devops-agent
objective: "Systematically diagnose unspecified Transcribe issues"
context: "Use this runbook when the issue doesn't clearly map to a specific runbook."
---

## Phase 1 — Triage

MUST:
- List failed jobs: `aws transcribe list-transcription-jobs --status FAILED --max-results 10`
- Check specific job: `aws transcribe get-transcription-job --transcription-job-name <name>`
- Verify IAM permissions
- Check audio file accessibility

SHOULD:
- Test with simple transcription job
- Check vocabularies and their status
- Review service quotas

MAY:
- Check service health dashboard

## Phase 2 — Remediate

MUST:
- Route to specific runbook once problem domain identified

SHOULD:
- Check multiple problem domains if initial investigation inconclusive

MAY:
- Engage AWS Support if issue persists

## Common Issues

- symptoms: "General Transcribe errors"
  diagnosis: "Start with job status, then audio format, then permissions."
  resolution: "Follow systematic triage. Route to specific runbook."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: api_errors
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook"
  long_term: "Implement comprehensive Transcribe monitoring"
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

- Audio recordings may contain private conversations
- Transcription output preserves all spoken content including PII
- Medical transcriptions contain protected health information (PHI)
- Classification: HIGHLY CONFIDENTIAL - audio content and potential PHI/PII

## Prohibited Actions

- NEVER process audio containing PII without content redaction enabled
- NEVER delete custom vocabularies used by production transcription jobs
- NEVER modify medical transcription settings without clinical team approval
- NEVER store transcription output containing sensitive data without encryption

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
