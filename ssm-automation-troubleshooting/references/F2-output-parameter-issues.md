---
title: "F2 — Output Parameter Issues"
description: "Diagnose output parameter passing failures between automation steps including reference syntax and type issues"
status: active
severity: MEDIUM
triggers: ["output empty", "parameter not passed", "step output missing", "variable reference error", "output parameter null"]
owner: devops-agent
objective: "Resolve output parameter passing issues between automation steps"
context: "Automation steps pass data via output parameters using {{stepName.outputKey}} syntax. All outputs are strings or StringList. Invalid references produce empty values silently — no error is thrown."
---

## Phase 1 — Triage

MUST:
- Get step execution details with outputs: `aws ssm describe-automation-step-executions --automation-execution-id <execution-id>`
- Check the producing step's actual output keys and values
- Verify the consuming step's input references match exactly

SHOULD:
- Review the document to compare output definitions with references: `aws ssm get-document --name <document-name>`
- Check that output key names are case-sensitive matches
- Verify the producing step completed successfully before the consuming step

MAY:
- Test with hardcoded values to isolate whether the issue is output passing or step logic
- Check if outputs exceed size limits (100KB per output)

## Phase 2 — Remediate

MUST:
- Fix output reference syntax: ensure {{stepName.outputKey}} matches exactly
- Verify producing step defines the output key in its outputs section
- Ensure producing step actually populates the output value

SHOULD:
- Use aws:executeScript for complex output transformations
- Serialize complex objects to JSON strings for passing between steps
- Add validation steps that check output values before critical operations

MAY:
- Implement default values for optional outputs
- Add logging to trace output values through the workflow

## Safety Ratings

```
safety_ratings:
  - "describe-automation-step-executions: GREEN — read-only step details"
  - "get-document: GREEN — read-only document content"
  - "update-document: YELLOW — modifies document to fix output definitions"
  - "start-automation-execution: YELLOW — re-runs automation to test fixes"
```

## Escalation Conditions

- Output parameter failure causes downstream steps to operate on wrong resources
- Output size exceeds 100KB limit and cannot be reduced
- Complex output transformations require significant document restructuring
- Output issues affect production automation pipelines

## Data Sensitivity

- `aws ssm describe-automation-step-executions`: MEDIUM — contains step output values
- `aws ssm get-document`: MEDIUM — contains document logic and parameter definitions
- Step outputs: MEDIUM to HIGH — may contain resource identifiers, configuration data
- `aws cloudtrail lookup-events`: MEDIUM — reveals execution parameters

## Prohibited Actions

- NEVER pass sensitive data (passwords, keys) as plain text output parameters
- NEVER assume output references work without testing — they fail silently
- NEVER ignore empty output values — they indicate a reference or step issue
- NEVER use output parameters for data larger than 100KB — use S3 or Parameter Store

## Phase 3 — Rollback

1. If incorrect output values caused wrong resources to be modified, identify affected resources
2. Review the consuming step's actions with the incorrect input values
3. Revert resource changes made with wrong parameter values
4. Fix the document output definitions and references
5. Re-test the automation with correct output passing before production use

## Common Issues

- symptoms: "Step receives empty string instead of expected output value"
  diagnosis: "The {{stepName.outputKey}} reference has a typo in step name or output key (case-sensitive)."
  resolution: "Verify exact step name and output key. Check the producing step's outputs section in the document."

- symptoms: "Output contains [object Object] instead of actual data"
  diagnosis: "A complex object was passed as output without serialization."
  resolution: "Serialize objects to JSON strings in the producing step. Parse JSON in the consuming step."

- symptoms: "Output truncated or missing data"
  diagnosis: "Output exceeds the 100KB size limit per output parameter."
  resolution: "Reduce output size. Store large data in S3 or Parameter Store and pass the reference instead."

## Output Format

```yaml
root_cause: "output_parameter — <specific_cause>"
evidence:
  - type: output_error
    content: "<expected vs actual output details>"
severity: MEDIUM
mitigation:
  immediate: "Fix output references"
  long_term: "Add output validation steps"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
