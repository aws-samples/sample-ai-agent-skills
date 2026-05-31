---
title: "H2 — Post-Upload Processing Issues"
description: "Diagnose issues with post-upload processing including Lambda steps, copy, tag, and delete operations"
status: active
severity: MEDIUM
triggers:
  - "post-upload"
  - "file processing"
  - "Lambda step"
  - "copy step"
  - "tag step"
  - "delete step"
  - "decrypt step"
owner: devops-agent
objective: "Resolve issues with individual workflow steps in post-upload processing"
context: "Managed workflow steps include: Copy (move files to a destination), Tag (add S3 object tags), Delete (remove the original file), Decrypt (PGP decryption), and Custom (Lambda function). Each step has specific IAM permission requirements and configuration parameters. Steps execute sequentially, and a failure in one step can halt the entire workflow."
---

## Phase 1 — Triage

MUST:
- Get workflow step details: `aws transfer describe-execution --execution-id <execution-id> --workflow-id <workflow-id>`
- Identify which step failed from the execution results
- Check the execution role permissions for the failing step type
- For Lambda custom steps, check Lambda function logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name>`
- Verify the step configuration (destination bucket, tag key/values, Lambda ARN)

SHOULD:
- Check if the source file still exists (may have been deleted by a previous step)
- Verify the destination bucket/prefix exists and is accessible
- For decrypt steps, verify the PGP key is correctly configured
- Check if the Lambda function has the correct handler and runtime

MAY:
- Test the Lambda function independently with a sample event
- Check if S3 event notifications are duplicating processing
- Verify no S3 bucket policies block the workflow execution role

## Phase 2 — Remediate

MUST:
- For copy step: ensure execution role has s3:GetObject (source), s3:PutObject (destination), and s3:GetObjectTagging if preserving tags
- For tag step: ensure execution role has s3:PutObjectTagging
- For delete step: ensure execution role has s3:DeleteObject
- For decrypt step: ensure PGP private key is correctly imported and execution role has required permissions
- For custom Lambda step: ensure execution role has lambda:InvokeFunction and the Lambda function is working

SHOULD:
- Add exception handler steps to gracefully handle failures
- Implement logging in custom Lambda steps for debugging
- Use step-level error handling to continue workflow on non-critical step failures

MAY:
- Implement dead-letter queues for failed Lambda invocations
- Add CloudWatch metrics and alarms for step-level failures
- Use S3 event notifications as a backup trigger for critical processing

## Common Issues

- symptoms: "Copy step fails — destination file not created"
  diagnosis: "Execution role lacks s3:PutObject on the destination bucket, or destination bucket does not exist."
  resolution: "Add s3:PutObject permission for the destination bucket ARN to the execution role."

- symptoms: "Custom Lambda step times out"
  diagnosis: "Lambda function processing takes longer than the configured timeout (max 5 minutes for workflow Lambda)."
  resolution: "Optimize Lambda function. For long processing, use Lambda to trigger an async process (Step Functions, SQS)."

- symptoms: "Decrypt step fails with 'invalid key' error"
  diagnosis: "PGP private key does not match the public key used to encrypt the file, or key is expired."
  resolution: "Verify the correct PGP private key is configured. Check key expiration. Re-import if necessary."

## Output Format

```yaml
root_cause: "post_upload_processing — <specific_cause>"
evidence:
  - type: execution_steps
    content: "<step-by-step execution results>"
  - type: failing_step
    content: "<specific step error details>"
  - type: permissions
    content: "<execution role permissions for the failing step>"
severity: MEDIUM
mitigation:
  immediate: "Fix the failing step configuration or permissions"
  long_term: "Add error handling and monitoring for each workflow step"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Transferred files may contain any type of sensitive data
- SSH keys and credentials enable server access
- Directory mappings reveal S3/EFS data organization
- Classification: CONFIDENTIAL - file transfer content and access credentials

## Prohibited Actions

- NEVER modify production server endpoints during active file transfers
- NEVER change authentication providers without user notification
- NEVER delete SSH host keys without coordinating with all SFTP clients
- NEVER modify directory mappings that could expose unauthorized S3/EFS paths

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest disabling SFTP encryption"
  - "NEVER suggest FTP protocol for production"
  - "NEVER suggest overly broad IAM roles for transfer users"
