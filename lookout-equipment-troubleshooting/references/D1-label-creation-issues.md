---
title: "D1 — Label Creation Issues"
description: "Diagnose Amazon Lookout for Equipment label group and label creation issues"
status: active
severity: MEDIUM
triggers: ["cannot create labels", "label group error", "label import failed", "CreateLabel error"]
owner: devops-agent
objective: "Resolve label creation issues to enable supervised model training"
context: "Labels mark known anomaly periods in sensor data. They improve model accuracy by providing supervised learning signals. Labels are organized in label groups."
---

## Phase 1 — Triage

MUST:
- List label groups: `aws lookoutequipment list-label-groups`
- Check label group details: `aws lookoutequipment describe-label-group --label-group-name <name>`
- List labels in the group: `aws lookoutequipment list-labels --label-group-name <name>`
- Verify IAM permissions for lookoutequipment:CreateLabel and CreateLabelGroup

SHOULD:
- Check that label time ranges fall within the dataset's data time range
- Verify label start times are before end times
- Check for overlapping labels that may cause conflicts

MAY:
- Review CloudTrail for label creation errors
- Check if the label group is associated with the correct dataset

## Phase 2 — Remediate

MUST:
- Create label group before creating labels: `aws lookoutequipment create-label-group --label-group-name <name>`
- Ensure label time ranges use ISO 8601 format
- Verify label start time is strictly before end time
- Ensure labels reference valid equipment and fault codes

SHOULD:
- Align label time ranges with actual known anomaly periods from maintenance records
- Avoid creating labels for periods outside the training data time range
- Use consistent fault codes across labels for the same type of anomaly

MAY:
- Batch import labels from maintenance management systems
- Create labels programmatically from CMMS (Computerized Maintenance Management System) exports

## Common Issues

- symptoms: "CreateLabel returns ValidationException about time range"
  diagnosis: "Label start time is after end time, or the time range format is invalid."
  resolution: "Ensure start_time is before end_time and both are in ISO 8601 format (e.g., 2024-01-15T00:00:00Z)."

- symptoms: "Labels created but model training does not improve"
  diagnosis: "Labels may not overlap with the training data time range, or too few labels were provided."
  resolution: "Verify labels fall within the training time range. Provide labels for multiple anomaly instances. AWS recommends at least 5 labeled anomaly periods."

- symptoms: "CreateLabelGroup returns ResourceAlreadyExistsException"
  diagnosis: "A label group with the same name already exists."
  resolution: "Use the existing label group or choose a different name. List existing groups with list-label-groups."

## Safety Ratings

```
safety_ratings:
  - "list-label-groups: GREEN — read-only enumeration"
  - "list-labels: GREEN — read-only label listing"
  - "CreateLabelGroup: GREEN — creates metadata container, no compute cost"
  - "CreateLabel: GREEN — creates metadata entry, no compute cost"
  - "delete-label: YELLOW — removes training signal, may affect future model quality"
```

## Escalation Conditions

- Label creation fails with internal service errors after format fixes
- Domain experts disagree on anomaly period boundaries for labeling
- Large volume of labels needs to be imported from external maintenance systems
- Labels are needed urgently for a model retraining deadline

## Data Sensitivity

- `aws lookoutequipment list-labels`: LOW — returns time ranges and fault codes
- Label time ranges: MEDIUM — may reveal equipment failure history and patterns
- Fault codes: MEDIUM — may contain proprietary equipment failure classifications
- Maintenance records used for labeling: HIGH — may contain operational details

## Prohibited Actions

- NEVER create labels without consulting equipment maintenance experts on anomaly boundaries
- NEVER delete labels from a label group that is actively used by a production model
- NEVER fabricate labels for time periods where anomaly status is unknown
- NEVER share label data externally — it reveals proprietary equipment failure patterns

## Phase 3 — Rollback

1. If incorrect labels were created, delete them: `aws lookoutequipment delete-label --label-group-name <name> --label-id <id>`
2. If a label group was created in error, delete all labels first, then the group
3. If labels affected a model retrain, the previous model version remains available
4. Document label corrections and the rationale for changes
5. Notify the ML team if label changes require model retraining

## Output Format

```yaml
root_cause: "label_creation — <specific_cause>"
evidence:
  - type: label_error
    content: "<error details from API response>"
severity: MEDIUM
mitigation:
  immediate: "Fix label format or permissions"
  long_term: "Automate label import from maintenance management systems"
```

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
