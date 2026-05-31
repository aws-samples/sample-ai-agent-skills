---
title: "E2 — Training Data Issues"
description: "Diagnose training data quality and format problems"
status: active
severity: MEDIUM
triggers:
  - "training data rejected"
  - "data quality error"
  - "insufficient training data"
owner: devops-agent
objective: "Resolve training data issues"
context: "Training data must meet format, quality, and size requirements."
---

## Phase 1 — Triage

MUST:
- Check training data format requirements
- Verify data size meets minimum
- Check for encoding issues
- Verify S3 access

SHOULD:
- Review data quality metrics
- Check for duplicate or low-quality files
- Verify language distribution

## Phase 2 — Remediate

MUST:
- Fix data format issues
- Add more data if below minimum
- Fix encoding to UTF-8

SHOULD:
- Curate high-quality training data
- Remove generated or test code

## Common Issues

- symptoms: "Customization rejects training data"
  diagnosis: "Data below minimum size or wrong format."
  resolution: "Add more code files. Ensure proper format."

## Output Format

```yaml
root_cause: "training_data — <specific_cause>"
evidence:
  - type: data_quality
    content: "<training data analysis>"
severity: MEDIUM
mitigation:
  immediate: "Fix data issues"
  long_term: "Maintain curated training dataset"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — training data involves proprietary code
- Service Disruption Risk: LOW — training data issues affect customization quality
- Compliance Impact: HIGH — training data must comply with data usage policies

## Escalation Conditions
- Training data contains sensitive or unauthorized code
- Training data quality affects suggestion quality
- Training data access permissions misconfigured
- Training data includes code from restricted repositories
- Training data retention exceeds approved periods

## Data Sensitivity
- Training code repositories — CONFIDENTIAL
- Training data configurations — CONFIDENTIAL
- Data access permissions — CONFIDENTIAL
- Training metrics — INTERNAL
- Never expose training data or repository details

## Prohibited Actions
- Do NOT include unauthorized code in training data
- Do NOT bypass training data governance review
- Do NOT share training data externally
- Do NOT retain training data beyond approved periods
- Do NOT train on code with incompatible licenses
- Do NOT ignore training data quality issues

## Phase 3 — Rollback
- Remove unauthorized training data
- Restore previous training configurations
- Revert data access permission changes
- Verify customization quality after changes
- Document training data issues
- Update training data governance procedures

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
