---
title: "C1 — Parallel Data Issues"
description: "Diagnose parallel data creation and import problems"
status: active
severity: MEDIUM
triggers:
  - "parallel data creation failed"
  - "CreateParallelData error"
  - "parallel data format error"
owner: devops-agent
objective: "Identify and resolve parallel data issues"
context: "Parallel data provides example translations for custom model training. Supports CSV, TMX, TSV formats. Data must be in S3. Training happens when used in batch jobs."
---

## Phase 1 — Triage

MUST:
- List parallel data: `aws translate list-parallel-data`
- Check parallel data: `aws translate get-parallel-data --name <name>`
- Verify data file format and S3 location
- Check parallel data status

SHOULD:
- Verify file format (CSV, TMX, TSV)
- Check record count and quality
- Verify language codes match intended pairs

MAY:
- Review import errors
- Check S3 permissions

## Phase 2 — Remediate

MUST:
- Use correct file format with proper language code headers
- Upload to S3 with proper permissions
- Provide sufficient parallel examples (100+ recommended)

SHOULD:
- Use high-quality, domain-specific parallel examples
- Validate data before import

MAY:
- Implement parallel data quality pipeline

## Common Issues

- symptoms: "Parallel data import fails"
  diagnosis: "File format incorrect or S3 access denied."
  resolution: "Fix file format. Check S3 bucket permissions."

- symptoms: "Parallel data imported but translation not improved"
  diagnosis: "Insufficient examples or not domain-relevant."
  resolution: "Add more domain-specific examples. Minimum 100+ recommended."

## Output Format

```yaml
root_cause: "parallel_data — <specific_cause>"
evidence:
  - type: parallel_data_status
    content: "<parallel data details>"
severity: MEDIUM
mitigation:
  immediate: "Fix data format or S3 permissions"
  long_term: "Build parallel data quality pipeline"
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
