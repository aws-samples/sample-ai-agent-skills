---
title: "B3 — Enrichment Lambda Failures"
description: "Diagnose enrichment function failures in EventBridge Pipes"
status: active
severity: MEDIUM
triggers:
  - "enrichment failed"
  - "enrichment lambda"
  - "enrichment error"
  - "enrichment timeout"
  - "transform failed"
owner: devops-agent
objective: "Identify and resolve enrichment function failures in Pipes"
context: "Pipe enrichment (Lambda, Step Functions, API Gateway, API Destination) transforms events between source and target. Enrichment failures can be caused by function errors, timeouts, permission issues, or payload format problems."
---

## Phase 1 — Triage

MUST:
- Describe the pipe for enrichment configuration: `aws pipes describe-pipe --name <pipe-name>`
- Check enrichment function health: `aws lambda get-function --function-name <function-name>`
- Check Lambda invocation errors: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify pipe execution role has lambda:InvokeFunction permission

SHOULD:
- Check Lambda function logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms> --limit 20`
- Verify the enrichment function returns the correct response format
- Check Lambda timeout and memory configuration
- Review DLQ for enrichment failure details

MAY:
- Test the enrichment function independently with sample events
- Check Lambda concurrency limits
- Verify enrichment input/output transformation

## Phase 2 — Remediate

MUST:
- Fix the enrichment function code or configuration
- Ensure the function returns events in the expected format
- Verify IAM permissions for the enrichment function

SHOULD:
- Increase Lambda timeout if function is timing out
- Add error handling in the enrichment function
- Configure DLQ for enrichment failures

MAY:
- Implement enrichment function monitoring
- Add input validation in the enrichment function

## Common Issues

- symptoms: "Enrichment function invoked but returns error"
  diagnosis: "Function code throws an exception processing the event payload."
  resolution: "Check Lambda logs for the error. Fix the function code. Test with sample events."

- symptoms: "Enrichment function times out"
  diagnosis: "Function execution exceeds the configured timeout."
  resolution: "Increase Lambda timeout. Optimize function code. Check external dependencies."

- symptoms: "Events not reaching target after enrichment"
  diagnosis: "Enrichment function returns empty or incorrectly formatted response."
  resolution: "Enrichment must return the events array. Verify response format matches expectations."

## Output Format

```yaml
root_cause: "enrichment_failure — <specific_cause>"
evidence:
  - type: pipe
    content: "<pipe enrichment configuration>"
  - type: lambda
    content: "<function status, logs, and errors>"
  - type: metrics
    content: "<invocation and error metrics>"
severity: MEDIUM
mitigation:
  immediate: "Fix enrichment function code or configuration"
  long_term: "Implement enrichment monitoring and error handling"
```


## Safety Ratings
```
safety_ratings:
  - "Describe pipe enrichment configuration: GREEN — read-only API call"
  - "Check Lambda function health: GREEN — read-only API call"
  - "Check Lambda invocation errors: GREEN — read-only monitoring query"
  - "Check Lambda function logs: GREEN — read-only log inspection"
  - "Fix enrichment function code: YELLOW — changes event transformation logic"
  - "Increase Lambda timeout: YELLOW — may increase processing cost and latency"
```

## Escalation Conditions
- Pipe serves production event processing pipeline
- Enrichment failures causing event delivery to stop
- Lambda function errors indicating code or dependency issues
- Enrichment timeout affecting end-to-end event processing latency
- Fix requires enrichment function code changes

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Lambda function logs: may contain event payload data"
    - "Enrichment function code: business transformation logic"
    - "Event payloads: business data being transformed"
  handling: "Lambda logs may contain sensitive event data. Restrict log access appropriately."
```

## Prohibited Actions
- NEVER deploy untested enrichment function code to production
- NEVER increase Lambda timeout beyond the pipe's maximum processing window
- NEVER remove enrichment from a pipe without understanding downstream target expectations
- NEVER modify enrichment response format without coordinating with target consumers

## Phase 3 — Rollback
- If enrichment function code was updated: deploy previous function version using Lambda aliases or version rollback
- If Lambda timeout was increased: revert to previous timeout value
- If Lambda memory was changed: restore previous memory configuration
- If enrichment was removed from pipe: re-add enrichment with original configuration
- If IAM permissions were changed: revert execution role to previous policy version

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
