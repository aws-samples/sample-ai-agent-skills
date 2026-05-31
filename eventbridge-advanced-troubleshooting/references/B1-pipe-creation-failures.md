---
title: "B1 — Pipe Creation Failures"
description: "Diagnose EventBridge Pipes creation and configuration failures"
status: active
severity: HIGH
triggers:
  - "pipe creation failed"
  - "cannot create pipe"
  - "pipe error"
  - "pipe validation"
owner: devops-agent
objective: "Identify the root cause of EventBridge Pipes creation failures"
context: "Pipe creation can fail due to invalid source/target configuration, IAM role issues, unsupported source types, enrichment configuration errors, or service quota limits. Pipes connect sources directly to targets without event buses."
---

## Phase 1 — Triage

MUST:
- Check if the pipe exists: `aws pipes describe-pipe --name <pipe-name>`
- Verify the IAM execution role: `aws iam get-role --role-name <role-name>`
- Check the source type is supported (SQS, Kinesis, DynamoDB Streams, MSK, MQ)
- Verify source and target ARNs are correct

SHOULD:
- Check CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=pipes.amazonaws.com --max-results 20`
- Verify source resource exists and is accessible
- Check target resource exists and is accessible
- Verify enrichment configuration if specified

MAY:
- Check service quotas for pipe count
- Verify KMS key if using encryption
- Review pipe tags and description

## Phase 2 — Remediate

MUST:
- Fix source/target ARN configuration
- Ensure IAM role trusts pipes.amazonaws.com and has source/target permissions
- Use a supported source type

SHOULD:
- Configure appropriate batch size and window for the source type
- Set up DLQ for failed event processing
- Test the pipe with a small batch before production

MAY:
- Set up CloudWatch alarms for pipe failures
- Use IaC for consistent pipe deployments

## Common Issues

- symptoms: "ValidationException — unsupported source type"
  diagnosis: "The specified source is not one of the five supported types."
  resolution: "Pipe sources must be SQS, Kinesis, DynamoDB Streams, MSK, or MQ."

- symptoms: "Pipe created but in FAILED state"
  diagnosis: "IAM role lacks permissions to read from source or write to target."
  resolution: "Add source-specific read permissions and target-specific write permissions to the role."

- symptoms: "Enrichment configuration error"
  diagnosis: "Enrichment Lambda ARN is invalid or the role lacks lambda:InvokeFunction permission."
  resolution: "Verify enrichment ARN. Add lambda:InvokeFunction to the pipe execution role."

## Output Format

```yaml
root_cause: "pipe_creation — <specific_cause>"
evidence:
  - type: pipe
    content: "<pipe configuration or error>"
  - type: iam_role
    content: "<execution role and permissions>"
  - type: source_target
    content: "<source and target resource status>"
severity: HIGH
mitigation:
  immediate: "Fix source, target, or role configuration"
  long_term: "Standardize pipe creation with IaC templates"
```


## Safety Ratings
```
safety_ratings:
  - "Describe pipe: GREEN — read-only API call"
  - "Verify IAM execution role: GREEN — read-only IAM inspection"
  - "Check CloudTrail for errors: GREEN — read-only audit log query"
  - "Fix source/target ARN configuration: GREEN — corrects pipe configuration"
  - "Update IAM execution role: YELLOW — changes access scope for the pipe"
  - "Delete and recreate pipe: YELLOW — brief gap in event processing"
```

## Escalation Conditions
- Pipe serves production event processing pipeline
- Pipe creation failure blocking application deployment
- IAM role changes affect multiple pipes or services
- Source or target resource in another account requiring cross-account coordination
- Service quota for pipes reached

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Source and target ARNs: infrastructure resource identifiers"
    - "IAM execution role: permission boundary for event processing"
    - "Enrichment configuration: data transformation logic"
    - "Filter criteria: business logic for event selection"
  handling: "Do not expose pipe configuration details or enrichment logic externally."
```

## Prohibited Actions
- NEVER use overly permissive IAM roles for pipe execution
- NEVER create pipes without DLQ configuration in production
- NEVER delete a running pipe without confirming downstream consumers can handle the gap
- NEVER modify pipe source configuration without understanding in-flight event impact

## Phase 3 — Rollback
- If pipe was created with wrong configuration: delete the pipe and recreate with correct settings
- If IAM execution role was modified: revert IAM policy to previous version
- If pipe was deleted: recreate with the original configuration from IaC or documentation
- If enrichment was misconfigured: update pipe with correct enrichment ARN and settings
- If source/target was changed: update pipe back to original source/target configuration

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
