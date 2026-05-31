---
title: "C2 — Inference Execution Failures"
description: "Diagnose Amazon Lookout for Equipment individual inference execution failures"
status: active
severity: HIGH
triggers: ["inference execution failed", "no inference results", "inference output missing", "execution status FAILED"]
owner: devops-agent
objective: "Resolve individual inference execution failures while the scheduler is running"
context: "Each inference scheduler run is an execution. Executions can fail independently due to missing input data, schema mismatches, or S3 access issues."
---

## Phase 1 — Triage

MUST:
- List failed executions: `aws lookoutequipment list-inference-executions --inference-scheduler-name <name> --status FAILED`
- Check scheduler is still running: `aws lookoutequipment describe-inference-scheduler --inference-scheduler-name <name>`
- Verify input data exists in S3 for the failed execution time window
- Check IAM role still has access to input and output S3 paths

SHOULD:
- Compare failed execution timestamps with data availability in S3
- Check if the input data format matches the training data schema
- Review the execution failure reason in the execution details

MAY:
- Check CloudWatch metrics for the inference scheduler
- Compare successful and failed execution time windows for data differences

## Phase 2 — Remediate

MUST:
- Ensure input data is written to S3 before each scheduled execution time
- Fix any schema mismatches between input data and the trained model
- Restore IAM role permissions if they were modified
- Verify S3 bucket policy has not changed to deny access

SHOULD:
- Implement data pipeline monitoring to ensure timely data delivery
- Set up CloudWatch alarms on consecutive execution failures
- Add data validation before writing to the inference input path

MAY:
- Adjust the data delay offset to account for pipeline latency
- Implement a dead-letter mechanism for failed execution data

## Common Issues

- symptoms: "Executions fail at the same time every day"
  diagnosis: "Data pipeline has a recurring delay or gap at that time, causing missing input data."
  resolution: "Investigate the upstream data pipeline schedule. Adjust the data delay offset or fix the pipeline timing."

- symptoms: "All executions fail after a model retrain"
  diagnosis: "The retrained model has a different schema or the inference scheduler was not updated to use the new model."
  resolution: "Verify the new model schema matches the input data. If the model name changed, recreate the scheduler with the new model."

- symptoms: "Executions fail with 'insufficient data' error"
  diagnosis: "The input data for the execution window has fewer rows than required for inference."
  resolution: "Ensure the data pipeline writes sufficient data points for each inference window. Check the sampling rate matches training data."

## Safety Ratings

```
safety_ratings:
  - "list-inference-executions: GREEN — read-only execution history"
  - "describe-inference-scheduler: GREEN — read-only scheduler status"
  - "s3 ls on input path: GREEN — read-only data verification"
  - "stop-inference-scheduler: YELLOW — stops monitoring, may miss anomalies"
  - "Modifying data pipeline: YELLOW — affects data flow to inference"
```

## Escalation Conditions

- More than 50% of executions fail over a 24-hour period
- Execution failures coincide with actual equipment anomalies being missed
- Failures persist after fixing data availability and IAM permissions
- Production equipment monitoring is degraded due to consecutive failures

## Data Sensitivity

- `aws lookoutequipment list-inference-executions`: LOW — returns execution metadata and status
- Inference input data: MEDIUM to HIGH — contains real-time sensor readings
- Inference output results: MEDIUM — contains anomaly scores and sensor contributions
- Execution failure reasons: LOW — contains error messages without raw data

## Prohibited Actions

- NEVER stop a production inference scheduler to troubleshoot individual execution failures
- NEVER delete inference execution history — it is needed for audit and pattern analysis
- NEVER modify the input data format without coordinating with the model training team
- NEVER ignore consecutive execution failures — they may indicate a systemic data pipeline issue

## Phase 3 — Rollback

1. If data pipeline changes caused failures, revert the pipeline configuration
2. If IAM changes caused failures, restore the previous role policy version
3. If a model update caused failures, update the scheduler to reference the previous model version
4. Verify executions resume successfully after the fix by monitoring the next 3-5 scheduled runs
5. Document the failure pattern and root cause for the operations runbook

## Output Format

```yaml
root_cause: "inference_execution — <specific_cause>"
evidence:
  - type: execution_failure
    content: "<failed execution details>"
severity: HIGH
mitigation:
  immediate: "Fix data availability or access issues"
  long_term: "Implement data pipeline monitoring and alerting"
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
