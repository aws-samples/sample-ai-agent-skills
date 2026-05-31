---
title: "F2 — Global Accelerator Cross-Account Endpoint Issues"
description: "Diagnose cross-account endpoint configuration problems"
status: active
severity: MEDIUM
triggers:
  - "cross-account endpoint"
  - "cross-account attachment"
  - "endpoint from another account"
owner: devops-agent
objective: "Resolve cross-account endpoint attachment and configuration issues"
context: "Global Accelerator supports endpoints in different AWS accounts using cross-account attachment resources. Both the accelerator account and endpoint account must take action."
---

## Phase 1 — Triage

MUST:
- List cross-account attachments: `aws globalaccelerator list-cross-account-attachments`
- Describe specific attachment: `aws globalaccelerator describe-cross-account-attachment --attachment-arn <attachment-arn>`
- List cross-account resources: `aws globalaccelerator list-cross-account-resources --resource-owner-aws-account-id <account-id>`
- Check endpoint group for cross-account endpoints: `aws globalaccelerator describe-endpoint-group --endpoint-group-arn <endpoint-group-arn>`

SHOULD:
- Verify the endpoint account has accepted the cross-account attachment
- Check IAM permissions in both accounts
- Verify the endpoint resource exists and is in the correct region

MAY:
- Check CloudTrail in both accounts for attachment-related events
- Verify Organizations policies allow cross-account Global Accelerator access

## Phase 2 — Remediate

MUST:
- Create cross-account attachment in the accelerator account: `aws globalaccelerator create-cross-account-attachment --name <name> --principals <endpoint-account-id>`
- Add the cross-account resource to the attachment
- Add the endpoint to the endpoint group using the cross-account resource ARN

SHOULD:
- Ensure IAM policies in the endpoint account allow Global Accelerator to access the resource
- Verify network connectivity between GA and the cross-account endpoint

MAY:
- Set up Organizations-level policies for cross-account GA access

## Common Issues

- symptoms: "Cannot add endpoint from another account"
  diagnosis: "Cross-account attachment not created or endpoint account not listed as principal."
  resolution: "Create cross-account attachment with the endpoint account as a principal."

- symptoms: "Cross-account endpoint shows as unhealthy"
  diagnosis: "Security group in endpoint account does not allow GA health check traffic."
  resolution: "Update security group in the endpoint account to allow health check traffic."

- symptoms: "Cross-account attachment creation fails"
  diagnosis: "IAM permissions missing for cross-account attachment operations."
  resolution: "Add globalaccelerator:CreateCrossAccountAttachment permission to the IAM policy."

## Output Format

```yaml
root_cause: "cross_account — <specific_cause>"
evidence:
  - type: attachment_config
    content: "<cross-account attachment details>"
  - type: endpoint_status
    content: "<cross-account endpoint health>"
  - type: iam_permissions
    content: "<IAM policy analysis>"
severity: MEDIUM
mitigation:
  immediate: "Create or fix cross-account attachment configuration"
  long_term: "Document cross-account setup and implement monitoring"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never disable or delete an accelerator serving production traffic without confirming failover
- Never remove endpoint groups without verifying traffic has been drained
- Never modify static IP associations on active accelerators
- Never change routing configurations during active incidents without approval
- Never delete flow logs that may be needed for audit or investigation

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Re-enable previous endpoint group configurations if traffic routing degrades
- Restore traffic dial settings to previous values if performance worsens
- Revert client affinity settings if session persistence breaks

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
