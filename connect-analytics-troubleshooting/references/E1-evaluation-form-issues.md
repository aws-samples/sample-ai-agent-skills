---
title: "E1 — Evaluation Form Issues"
description: "Diagnose Amazon Connect evaluation form configuration and submission issues"
status: active
severity: MEDIUM
triggers: ["evaluation form error", "cannot create form", "form not available", "evaluation submission failed"]
owner: devops-agent
objective: "Resolve evaluation form configuration and usage issues"
context: "Evaluation forms are used by supervisors and quality analysts to evaluate agent performance on specific contacts. Forms use Contact Lens data (transcript, sentiment) as input. Forms must be activated before use."
---

## Phase 1 — Triage

MUST:
- List evaluation forms: `aws connect list-evaluation-forms --instance-id <instance-id>`
- Get form details: `aws connect describe-evaluation-form --instance-id <instance-id> --evaluation-form-id <form-id>`
- Verify the form is in ACTIVE status

SHOULD:
- Check if the evaluator has permissions to submit evaluations
- Verify the contact has Contact Lens data available (transcript, sentiment)
- Check if the form version is current

MAY:
- Review CloudTrail for evaluation form creation and update events
- Check if the issue affects specific forms or all forms

## Phase 2 — Remediate

MUST:
- Activate the evaluation form if it is in draft status
- Verify form sections and questions are properly configured
- Ensure evaluators have connect:SubmitContactEvaluation permissions

SHOULD:
- Test the form with a sample evaluation before deploying to all evaluators
- Verify scoring criteria are correctly configured
- Ensure Contact Lens is enabled for contacts that will be evaluated

MAY:
- Create form templates for different evaluation scenarios
- Set up automated evaluation triggers using Contact Lens categories
- Configure evaluation reporting and analytics

## Safety Ratings

```
safety_ratings:
  - "list-evaluation-forms: GREEN — read-only form enumeration"
  - "describe-evaluation-form: GREEN — read-only form details"
  - "activate-evaluation-form: YELLOW — makes form available to evaluators"
  - "update-evaluation-form: YELLOW — modifies form structure"
  - "delete-evaluation-form: RED — removes form and associated evaluations"
```

## Escalation Conditions

- Evaluation form activation fails with validation errors
- Submitted evaluations are lost or not saved
- Form scoring produces incorrect results
- Compliance requires specific evaluation criteria that cannot be configured

## Data Sensitivity

- `aws connect list-evaluation-forms`: LOW — returns form metadata
- `aws connect describe-evaluation-form`: MEDIUM — reveals evaluation criteria and scoring
- Evaluation submissions: HIGH — contain agent performance assessments
- Contact data used in evaluations: HIGH — contains customer interaction details

## Prohibited Actions

- NEVER delete evaluation forms that have submitted evaluations without archiving
- NEVER modify active evaluation forms without versioning
- NEVER access evaluation results without proper authorization
- NEVER use evaluation forms as the sole basis for disciplinary action without human review

## Phase 3 — Rollback

1. Deactivate the form if activation caused issues
2. Revert form changes by restoring the previous version
3. If evaluations were submitted against a misconfigured form, document the issue for re-evaluation
4. Verify evaluators can access the correct form version after changes
5. Document the form change and rollback reason

## Common Issues

- symptoms: "Evaluation form shows 'No transcript available' for a contact"
  diagnosis: "Contact Lens was not enabled for the contact — no transcript was generated."
  resolution: "Ensure Contact Lens is enabled in the contact flow. Evaluations require Contact Lens data."

## Output Format

```yaml
root_cause: "evaluation_form — <specific_cause>"
evidence:
  - type: form_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix evaluation form configuration"
  long_term: "Implement form versioning and testing procedures"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
