---
title: "C1 — Inference Scheduler Errors"
description: "Diagnose Amazon Lookout for Equipment inference scheduler creation and configuration errors"
status: active
severity: HIGH
triggers: ["inference scheduler failed", "cannot create scheduler", "scheduler not starting", "CreateInferenceScheduler error"]
owner: devops-agent
objective: "Resolve inference scheduler creation and configuration issues"
context: "Inference schedulers run anomaly detection on a fixed cadence (5 min to 1 hour). They require a trained model, S3 input/output paths, and proper IAM permissions."
---

## Phase 1 — Triage

MUST:
- Check scheduler status: `aws lookoutequipment describe-inference-scheduler --inference-scheduler-name <name>`
- Verify the model is in ACTIVE status: `aws lookoutequipment describe-model --model-name <name>`
- Check IAM role permissions for S3 input and output paths
- Verify the data upload frequency matches the scheduler interval

SHOULD:
- Check CloudTrail for CreateInferenceScheduler errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateInferenceScheduler --max-results 10`
- Verify S3 input path structure matches expected format
- Check service quotas for maximum inference schedulers

MAY:
- Review the input configuration for timestamp format and delimiter settings
- Check if the model was trained with compatible data format

## Phase 2 — Remediate

MUST:
- Ensure the model is in ACTIVE status before creating a scheduler
- Fix IAM role with s3:GetObject on input path and s3:PutObject on output path
- Set data upload frequency to match or exceed the scheduler interval
- Verify input S3 path contains data in the expected format

SHOULD:
- Configure the data delay offset to account for data pipeline latency
- Set up CloudWatch alarms for scheduler execution failures
- Test with a short interval first before production deployment

MAY:
- Use EventBridge to trigger notifications on scheduler state changes
- Implement S3 event notifications to confirm data availability before scheduled runs

## Common Issues

- symptoms: "CreateInferenceScheduler returns ValidationException about model status"
  diagnosis: "The specified model is not in ACTIVE status. It may still be training or may have failed."
  resolution: "Wait for model training to complete. Check model status with describe-model. Only ACTIVE models can be used for inference."

- symptoms: "Scheduler created but immediately goes to STOPPED status"
  diagnosis: "The scheduler was created with an invalid configuration that prevents it from starting."
  resolution: "Check the input/output S3 configuration, IAM role, and data upload frequency. Delete and recreate with corrected parameters."

- symptoms: "Scheduler is RUNNING but no output appears in S3"
  diagnosis: "The scheduler is running but no input data is available at the scheduled times, or the output path is incorrect."
  resolution: "Verify data is being written to the input S3 path before each scheduled interval. Check the output S3 path configuration."

## Safety Ratings

```
safety_ratings:
  - "describe-inference-scheduler: GREEN — read-only status check"
  - "CreateInferenceScheduler: YELLOW — creates a running resource with ongoing compute costs"
  - "start-inference-scheduler: YELLOW — starts compute-consuming inference runs"
  - "stop-inference-scheduler: YELLOW — stops inference, may cause monitoring gaps"
  - "delete-inference-scheduler: RED — permanently removes scheduler and stops all inference"
```

## Escalation Conditions

- Scheduler creation fails with internal service errors after configuration fixes
- Scheduler runs but produces incorrect or empty results consistently
- Multiple schedulers needed for the same model with different configurations
- Production monitoring gap due to scheduler downtime exceeding the SLA

## Data Sensitivity

- `aws lookoutequipment describe-inference-scheduler`: LOW — returns scheduler configuration
- Inference input data: MEDIUM to HIGH — contains real-time sensor telemetry
- Inference output data: MEDIUM — contains anomaly detection results and sensor contributions
- S3 paths: LOW — may reveal project or environment naming conventions

## Prohibited Actions

- NEVER delete an inference scheduler that is actively monitoring production equipment without explicit approval
- NEVER suggest reducing the inference interval below what the data pipeline can support
- NEVER create multiple schedulers for the same model without understanding the cost implications
- NEVER modify the S3 output path of a running scheduler without updating downstream consumers

## Phase 3 — Rollback

1. If a scheduler was created with wrong configuration, stop it: `aws lookoutequipment stop-inference-scheduler --inference-scheduler-name <name>`
2. Delete the misconfigured scheduler: `aws lookoutequipment delete-inference-scheduler --inference-scheduler-name <name>`
3. Recreate with corrected parameters after verifying model, IAM, and S3 configuration
4. Verify the new scheduler produces expected output by checking the first execution
5. Notify the operations team of any monitoring gaps during scheduler recreation

## Output Format

```yaml
root_cause: "inference_scheduler — <specific_cause>"
evidence:
  - type: scheduler_error
    content: "<error details from describe-inference-scheduler>"
severity: HIGH
mitigation:
  immediate: "Fix configuration and recreate scheduler"
  long_term: "Implement scheduler health monitoring with CloudWatch alarms"
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
