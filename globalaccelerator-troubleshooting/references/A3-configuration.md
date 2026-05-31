---
title: "A3 — Global Accelerator Configuration Issues"
description: "Diagnose accelerator configuration problems including IP type and attributes"
status: active
severity: MEDIUM
triggers:
  - "accelerator misconfigured"
  - "configuration error"
  - "accelerator attributes"
owner: devops-agent
objective: "Identify and resolve Global Accelerator configuration issues"
context: "Accelerator configuration includes IP address type (IPv4/dual-stack), flow logs, and enabled state. Misconfiguration can cause traffic routing failures."
---

## Phase 1 — Triage

MUST:
- Get full accelerator configuration: `aws globalaccelerator describe-accelerator --accelerator-arn <accelerator-arn>`
- Check accelerator attributes (flow logs): `aws globalaccelerator describe-accelerator-attributes --accelerator-arn <accelerator-arn>`
- List all listeners: `aws globalaccelerator list-listeners --accelerator-arn <accelerator-arn>`
- Check recent configuration changes: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=globalaccelerator.amazonaws.com --max-results 10`

SHOULD:
- Verify IP address type matches requirements (IPV4 vs DUAL_STACK)
- Check if flow logs are enabled and S3 bucket is accessible
- Verify tags are correct: `aws globalaccelerator list-tags-for-resource --resource-arn <accelerator-arn>`

MAY:
- Compare configuration with a known-good accelerator
- Check if the accelerator is associated with Shield Advanced

## Phase 2 — Remediate

MUST:
- Update accelerator configuration as needed: `aws globalaccelerator update-accelerator --accelerator-arn <accelerator-arn> --enabled`
- Fix flow log S3 bucket permissions if flow logs are failing

SHOULD:
- Enable flow logs for troubleshooting: `aws globalaccelerator update-accelerator-attributes --accelerator-arn <accelerator-arn> --flow-logs-enabled --flow-logs-s3-bucket <bucket> --flow-logs-s3-prefix <prefix>`

MAY:
- Update IP address type if dual-stack is needed

## Common Issues

- symptoms: "Flow logs not appearing in S3"
  diagnosis: "S3 bucket policy does not allow Global Accelerator to write logs."
  resolution: "Update S3 bucket policy to allow globalaccelerator.amazonaws.com to PutObject."

- symptoms: "IPv6 clients cannot connect"
  diagnosis: "Accelerator IP address type is set to IPV4 only."
  resolution: "Update accelerator to DUAL_STACK IP address type."

- symptoms: "Configuration changes not taking effect"
  diagnosis: "Accelerator is still in IN_PROGRESS state after update."
  resolution: "Wait for accelerator to reach DEPLOYED status before testing."

## Output Format

```yaml
root_cause: "configuration_issue — <specific_cause>"
evidence:
  - type: accelerator_config
    content: "<full accelerator configuration>"
  - type: attributes
    content: "<accelerator attributes>"
  - type: cloudtrail
    content: "<recent configuration changes>"
severity: MEDIUM
mitigation:
  immediate: "Update accelerator configuration"
  long_term: "Implement configuration monitoring and change tracking"
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
