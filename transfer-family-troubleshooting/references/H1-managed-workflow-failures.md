---
title: "H1 — Managed Workflow Execution Failures"
description: "Diagnose Transfer Family managed workflow execution failures"
status: active
severity: HIGH
triggers:
  - "workflow failed"
  - "workflow execution"
  - "managed workflow"
  - "workflow error"
  - "post-upload workflow"
  - "workflow not triggered"
owner: devops-agent
objective: "Identify and resolve managed workflow execution failures"
context: "Transfer Family managed workflows execute automatically on file upload (or optionally on partial upload). Workflows can include copy, tag, decrypt, delete, and custom Lambda steps. Failures can occur due to IAM permissions, Lambda errors, step configuration, or the workflow not being associated with the server."
---

## Phase 1 — Triage

MUST:
- Check workflow association on the server: `aws transfer describe-server --server-id <server-id> --query 'Server.WorkflowDetails'`
- List recent workflow executions: `aws transfer list-executions --workflow-id <workflow-id> --query 'Executions[*].{ExecutionId:ExecutionId,Status:Status,InitialFileLocation:InitialFileLocation}'`
- Get execution details for failed executions: `aws transfer describe-execution --execution-id <execution-id> --workflow-id <workflow-id>`
- Check the workflow execution role permissions: `aws iam get-role --role-name <execution-role>`
- Check structured logs for workflow errors: `aws logs filter-log-events --log-group-name /aws/transfer/<server-id> --filter-pattern "\"workflow\""`

SHOULD:
- Verify the workflow execution role trusts transfer.amazonaws.com
- Check if the workflow has both OnUpload and OnPartialUpload handlers configured
- Verify Lambda functions in custom steps are accessible and not erroring
- Check if the workflow was recently modified (CloudTrail)

MAY:
- Test the workflow by uploading a small test file
- Check Lambda function logs for custom step errors
- Verify S3 event notifications are not conflicting with workflow triggers

## Phase 2 — Remediate

MUST:
- Associate the workflow with the server if not already: OnUpload and/or OnPartialUpload
- Ensure the workflow execution role has permissions for all workflow steps (S3 copy, S3 tag, Lambda invoke, etc.)
- Fix Lambda function errors in custom steps
- Ensure the execution role trusts transfer.amazonaws.com

SHOULD:
- Add error handling steps in the workflow (exception handlers)
- Monitor workflow executions with CloudWatch alarms on failure count
- Use OnPartialUpload workflows for handling interrupted transfers

MAY:
- Implement idempotent workflow steps to handle retries safely
- Add SNS notifications for workflow failures

## Common Issues

- symptoms: "Workflow not triggered after file upload"
  diagnosis: "Workflow not associated with the server, or associated with OnPartialUpload instead of OnUpload."
  resolution: "Associate the workflow with the server's OnUpload trigger."

- symptoms: "Workflow execution fails at custom Lambda step"
  diagnosis: "Lambda function throws an error, times out, or the execution role lacks lambda:InvokeFunction permission."
  resolution: "Check Lambda logs. Add lambda:InvokeFunction to the execution role. Increase Lambda timeout."

- symptoms: "Workflow copy step fails with access denied"
  diagnosis: "Execution role lacks s3:GetObject on the source or s3:PutObject on the destination."
  resolution: "Add required S3 permissions to the workflow execution role for both source and destination."

## Output Format

```yaml
root_cause: "workflow_failure — <specific_cause>"
evidence:
  - type: workflow_config
    content: "<workflow association and steps>"
  - type: execution_details
    content: "<execution status and error details>"
  - type: execution_role
    content: "<role permissions>"
severity: HIGH
mitigation:
  immediate: "Fix workflow configuration or execution role permissions"
  long_term: "Add error handling, monitoring, and alerting for workflow executions"
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
