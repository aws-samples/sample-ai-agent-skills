---
title: "D1 — Custom Labels Project Issues"
description: "Diagnose Custom Labels project creation and dataset problems"
status: active
severity: HIGH
triggers:
  - "custom labels project error"
  - "dataset creation failed"
  - "project creation error"
  - "manifest file error"
owner: devops-agent
objective: "Identify and resolve Custom Labels project and dataset issues"
context: "Custom Labels projects contain datasets (training/test) and model versions. Datasets use manifest files or console labeling. S3 permissions required for image access."
---

## Phase 1 — Triage

MUST:
- List projects: `aws rekognition describe-projects`
- Check project status: `aws rekognition describe-projects --project-names <project-name>`
- Verify dataset: `aws rekognition describe-dataset --dataset-arn <dataset-arn>`
- Check S3 access for training images

SHOULD:
- Verify manifest file format (SageMaker Ground Truth format)
- Check minimum image count per label
- Verify image formats in dataset

MAY:
- Review dataset statistics
- Check for labeling errors

## Phase 2 — Remediate

MUST:
- Fix manifest file format
- Ensure S3 bucket accessible to Rekognition
- Provide minimum training images per label (10+)

SHOULD:
- Split data into training and test datasets
- Use diverse images for each label

MAY:
- Implement dataset validation pipeline

## Common Issues

- symptoms: "Dataset creation fails with manifest error"
  diagnosis: "Manifest file format incorrect."
  resolution: "Use SageMaker Ground Truth manifest format. Verify JSON lines."

- symptoms: "Project shows insufficient training data"
  diagnosis: "Too few images per label."
  resolution: "Add more labeled images. Minimum 10 per label recommended."

## Output Format

```yaml
root_cause: "custom_labels_project — <specific_cause>"
evidence:
  - type: project_status
    content: "<project details>"
  - type: dataset_info
    content: "<dataset statistics>"
severity: HIGH
mitigation:
  immediate: "Fix dataset or manifest configuration"
  long_term: "Implement dataset quality validation"
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

- Images and video may contain biometric data (faces, persons)
- Analysis results may contain personally identifiable information
- Custom Labels training data may contain proprietary content
- Classification: HIGHLY CONFIDENTIAL - biometric and PII data requires special handling

## Prohibited Actions

- NEVER delete face collections containing production data
- NEVER stop running Custom Labels models without confirming downstream impact
- NEVER modify content moderation thresholds in production without approval
- NEVER process PII-containing images without confirming data handling compliance

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
