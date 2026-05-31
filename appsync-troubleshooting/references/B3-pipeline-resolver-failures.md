---
title: "B3 — Pipeline Resolver Failures"
description: "Diagnose pipeline resolver function chain failures"
status: active
severity: HIGH
triggers:
  - "pipeline resolver"
  - "pipeline function"
  - "function chain"
  - "ctx.prev.result"
  - "stash"
  - "pipeline error"
owner: devops-agent
objective: "Identify and resolve failures in AppSync pipeline resolver function chains"
context: "Pipeline resolvers chain multiple functions: before mapping → function1 → function2 → ... → after mapping. Each function has its own request/response handlers and data source. Data passes between functions via ctx.stash and ctx.prev.result. Failures can occur in any function in the chain."
---

## Phase 1 — Triage

MUST:
- Get the pipeline resolver: `aws appsync get-resolver --api-id <api-id> --type-name <type> --field-name <field>`
- List pipeline functions: `aws appsync list-functions --api-id <api-id>`
- Get each function in the pipeline: `aws appsync get-function --api-id <api-id> --function-id <function-id>`
- Enable field-level logging (ALL) to trace function execution order
- Check CloudWatch Logs for which function in the pipeline fails

SHOULD:
- Verify function execution order matches the intended data flow
- Check that ctx.stash is used correctly to pass data between functions
- Verify each function's data source is correctly configured

MAY:
- Test individual functions in isolation using the evaluate-code API
- Check if the before/after mapping templates correctly initialize/finalize the pipeline

## Phase 2 — Remediate

MUST:
- Fix the specific failing function in the pipeline
- Ensure data dependencies between functions are satisfied (ctx.prev.result, ctx.stash)
- Verify function order matches the required data flow

SHOULD:
- Add error handling in each function to provide clear error messages
- Use ctx.stash to pass data that multiple downstream functions need
- Keep pipeline functions focused on a single responsibility

MAY:
- Consider splitting complex pipelines into separate resolvers
- Add logging within functions for debugging

## Common Issues

- symptoms: "ctx.prev.result is null in function 2"
  diagnosis: "Function 1's response handler doesn't return a value, or returns undefined."
  resolution: "Ensure each function's response handler returns the data needed by the next function."

- symptoms: "Pipeline resolver returns partial data"
  diagnosis: "The after mapping template doesn't aggregate results from all functions correctly."
  resolution: "Use ctx.stash to accumulate data across functions and return it in the after mapping."

- symptoms: "Function executes with wrong data source"
  diagnosis: "Function is associated with the wrong data source."
  resolution: "Update the function's data source association: aws appsync update-function."

## Output Format

```yaml
root_cause: "pipeline_failure — <specific_cause>"
evidence:
  - type: pipeline_config
    content: "<pipeline function chain and order>"
  - type: function_logs
    content: "<individual function execution logs>"
severity: HIGH
mitigation:
  immediate: "Fix the failing function in the pipeline"
  long_term: "Add per-function error handling and logging"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix failing function in pipeline | YELLOW | Resolver function code change |
| Ensure data dependencies satisfied | YELLOW | Function data flow change |
| Verify function order | GREEN | Configuration verification |
| Add error handling in each function | YELLOW | Function code change |
| Use ctx.stash for data passing | YELLOW | Function code change |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **Pipeline configuration**: Function chain and order — INTERNAL
- **Function logs**: Individual function execution details — INTERNAL
- **Stash data**: Inter-function data passing — INTERNAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER reorder pipeline functions without understanding data dependencies
- NEVER deploy pipeline changes without testing each function individually

## Phase 3 — Rollback

1. Revert the failing function to its previous code version
2. Restore previous function order if it was changed
3. Revert data source associations if they were modified
4. Test the pipeline with known inputs after rollback
5. Verify all functions in the chain execute successfully

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
  - "NEVER suggest removing authentication from APIs"
  - "NEVER suggest API_KEY auth for production"
  - "NEVER suggest disabling logging"
