---
title: "A2 — Step Execution Errors"
description: "Diagnose individual step failures within SSM Automation executions"
status: active
severity: HIGH
triggers: ["step failed", "step timeout", "step error", "automation step not completing"]
owner: devops-agent
objective: "Resolve individual step execution errors within automation workflows"
context: "Each automation step has its own type, timeout, and failure handling. Steps can fail independently while the overall execution continues based on onFailure settings."
---

## Phase 1 — Triage

MUST:
- Get step execution details: `aws ssm describe-automation-step-executions --automation-execution-id <execution-id>`
- Identify the failed step name, type, and failure message
- Check step inputs and outputs for the failed step

SHOULD:
- Review the document definition for the failed step: `aws ssm get-document --name <document-name> --document-version <version>`
- Check if the step has onFailure or timeoutSeconds configured
- Verify step-specific IAM permissions

MAY:
- Check CloudWatch Logs if the step uses aws:executeScript with logging
- Review Lambda function logs if the step uses aws:invokeLambdaFunction

## Phase 2 — Remediate

MUST:
- Fix the specific step error based on step type (runCommand, executeScript, invokeLambda, etc.)
- Ensure step timeout is sufficient for the operation
- Verify step input parameters reference valid outputs from previous steps

SHOULD:
- Add error handling with onFailure and maxAttempts in the document
- Configure appropriate timeoutSeconds for long-running steps
- Validate output parameter references use correct {{stepName.outputKey}} syntax

MAY:
- Add aws:executeScript validation steps before critical operations
- Implement step-level retry logic with maxAttempts and backoff

## Safety Ratings

```
safety_ratings:
  - "describe-automation-step-executions: GREEN — read-only step details"
  - "get-document: GREEN — read-only document content"
  - "update-document: YELLOW — modifies automation document, affects future executions"
  - "start-automation-execution: YELLOW — re-running may repeat resource modifications"
```

## Escalation Conditions

- Step fails with internal service errors not related to permissions or parameters
- aws:executeScript step fails with runtime errors in custom Python/PowerShell code
- Step timeout cannot be increased due to upstream dependencies
- Multiple steps fail in sequence suggesting a systemic issue

## Data Sensitivity

- `aws ssm describe-automation-step-executions`: MEDIUM — may contain step outputs with resource data
- `aws ssm get-document`: MEDIUM — contains automation logic and parameter definitions
- `aws logs get-log-events`: MEDIUM — may contain application data from script execution
- `aws lambda get-function`: LOW — returns function metadata

## Prohibited Actions

- NEVER modify a running automation's document to fix a step mid-execution
- NEVER skip failed steps by modifying onFailure to Continue without understanding impact
- NEVER remove timeout configurations to avoid timeout errors
- NEVER re-run failed automations on production without understanding what steps already completed

## Phase 3 — Rollback

1. Identify which steps completed successfully before the failure
2. Review resource changes made by completed steps
3. If the document has rollback steps, trigger them manually if needed
4. For aws:runCommand steps, check target instance state and revert changes
5. Document partial execution state for recovery planning

## Common Issues

- symptoms: "aws:executeScript step fails with ScriptFailed"
  diagnosis: "Python or PowerShell script has a runtime error or returns incorrect output format."
  resolution: "Check script syntax. Ensure the script returns output in the expected format. Test script independently."

- symptoms: "Step fails with StepExecutionTimedOut"
  diagnosis: "The step exceeded its timeoutSeconds value."
  resolution: "Increase timeoutSeconds in the document. Check if the underlying operation is genuinely slow or stuck."

- symptoms: "Step output reference returns empty value"
  diagnosis: "The {{stepName.outputKey}} reference has incorrect step name or output key."
  resolution: "Verify exact step name and output key. References are case-sensitive. Check previous step actually produces the output."

## Output Format

```yaml
root_cause: "step_execution — <step_name> — <specific_cause>"
evidence:
  - type: step_error
    content: "<step failure message and details>"
severity: HIGH
mitigation:
  immediate: "Fix step-level error"
  long_term: "Add step validation and error handling"
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
