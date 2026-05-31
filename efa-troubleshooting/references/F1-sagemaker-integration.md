---
title: "F1 — SageMaker Integration"
description: "Diagnose EFA integration issues with SageMaker training jobs"
status: active
severity: HIGH
triggers: ["sagemaker efa error", "distributed training slow", "sagemaker efa not working", "training job efa"]
owner: devops-agent
objective: "Resolve EFA configuration issues in SageMaker distributed training jobs"
context: "SageMaker supports EFA for distributed training on supported instance types. EFA is configured through the training job resource configuration."
---

## Phase 1 — Triage

MUST:
- Check training job configuration: `aws sagemaker describe-training-job --training-job-name <name> --query '{InstanceType:ResourceConfig.InstanceType,InstanceCount:ResourceConfig.InstanceCount}'`
- Verify instance type supports EFA: `aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[0].NetworkInfo.EfaSupported'`
- Check training job status and failure reason: `aws sagemaker describe-training-job --training-job-name <name> --query '{Status:TrainingJobStatus,FailureReason:FailureReason}'`

SHOULD:
- Review training job logs in CloudWatch: `aws logs get-log-events --log-group-name /aws/sagemaker/TrainingJobs --log-stream-name <job-name>/algo-1-<timestamp>`
- Verify the training container includes EFA libraries

MAY:
- Compare performance with and without EFA to quantify benefit
- Check SageMaker service quotas for the instance type

## Phase 2 — Remediate

MUST:
- Enable EFA in the training job configuration: set `EnableNetworkIsolation=False` and use EFA-supported instance types
- Use SageMaker's built-in distributed training frameworks (SageMaker Distributed Data Parallel or Model Parallel)
- Ensure the training container has EFA drivers and NCCL with aws-ofi-nccl

SHOULD:
- Use SageMaker's pre-built deep learning containers which include EFA support
- Set NCCL environment variables in the training job hyperparameters or container environment

MAY:
- Build a custom training container with specific EFA/NCCL versions

## Safety Ratings

```
safety_ratings:
  - "sagemaker describe-training-job: GREEN — read-only job status query"
  - "ec2 describe-instance-types: GREEN — read-only instance type query"
  - "logs get-log-events: GREEN — read-only log retrieval"
  - "sagemaker create-training-job: YELLOW — launches training resources with cost"
```

## Escalation Conditions

- EFA is enabled but training performance matches non-EFA baseline
- SageMaker training job fails during NCCL initialization
- Custom container cannot access EFA devices inside SageMaker
- Instance type quota prevents launching sufficient EFA instances

## Data Sensitivity

- Training job configuration: MEDIUM — reveals model architecture and hyperparameters
- CloudWatch training logs: HIGH — may contain training data samples or model details
- Instance type and count: LOW — infrastructure configuration
- SageMaker quotas: LOW — account limits

## Prohibited Actions

- NEVER enable network isolation when EFA is required — it blocks EFA communication
- NEVER use non-EFA instance types and expect distributed training to scale linearly
- NEVER modify SageMaker-managed security groups directly
- NEVER share training job logs containing model details or training data

## Phase 3 — Rollback

1. If training job configuration was changed, revert to previous job definition
2. If custom container was deployed, roll back to the previous container image
3. Cancel running training jobs if they are failing: `aws sagemaker stop-training-job --training-job-name <name>`
4. Verify the previous training configuration still works
5. Document the EFA configuration that caused the issue

## Common Issues

- symptoms: "SageMaker distributed training is not faster than single-instance"
  diagnosis: "EFA is not enabled or the instance type does not support EFA."
  resolution: "Use EFA-supported instance types (p4d, p5, trn1). Verify EFA is enabled in resource config."

- symptoms: "Training job fails with NCCL initialization error"
  diagnosis: "Training container does not include aws-ofi-nccl plugin or EFA libraries."
  resolution: "Use SageMaker pre-built DLC images which include EFA support, or add EFA libraries to custom container."

## Output Format

```yaml
root_cause: "sagemaker_efa — <specific_cause>"
evidence:
  - type: training_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix EFA configuration in training job"
  long_term: "Standardize EFA-enabled training pipeline"
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
