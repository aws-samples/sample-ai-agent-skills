---
title: "A1 — Global Accelerator Creation Failures"
description: "Diagnose why a Global Accelerator accelerator fails to create"
status: active
severity: HIGH
triggers:
  - "accelerator creation failed"
  - "cannot create accelerator"
  - "create accelerator error"
owner: devops-agent
objective: "Identify and resolve Global Accelerator creation failures"
context: "Accelerator creation can fail due to IAM permissions, service limits, or invalid configuration. Global Accelerator is a global service managed from us-west-2 API endpoint."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions for the caller: `aws sts get-caller-identity`
- Verify service quotas: `aws service-quotas get-service-quota --service-code globalaccelerator --quota-code L-6E2D2BBC`
- List existing accelerators to check limits: `aws globalaccelerator list-accelerators --query 'Accelerators[*].{Name:Name,Status:Status,Arn:AcceleratorArn}'`
- Check CloudTrail for creation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateAccelerator --max-results 5`

SHOULD:
- Verify the IAM policy includes `globalaccelerator:CreateAccelerator` permission
- Check for SCPs that may block Global Accelerator actions
- Verify BYOIP address pool if using custom IPs: `aws globalaccelerator list-byoip-cidrs`

MAY:
- Check AWS Health Dashboard for Global Accelerator service issues
- Review Organizations SCPs if in a multi-account environment

## Phase 2 — Remediate

MUST:
- Ensure IAM principal has `globalaccelerator:CreateAccelerator` and `globalaccelerator:TagResource` permissions
- Request quota increase if at the accelerator limit (default: 20 per account)
- Verify BYOIP CIDR is in ADVERTISING state if using custom IPs

SHOULD:
- Use the correct API endpoint (Global Accelerator API is in us-west-2)
- Ensure tags comply with any tag policies in Organizations

MAY:
- Create the accelerator with IP address type set to IPV4 or DUAL_STACK based on requirements

## Common Issues

- symptoms: "AccessDeniedException when calling CreateAccelerator"
  diagnosis: "IAM policy missing globalaccelerator:CreateAccelerator permission."
  resolution: "Add the required IAM permissions or attach the GlobalAcceleratorFullAccess managed policy."

- symptoms: "LimitExceededException for accelerators"
  diagnosis: "Account has reached the maximum number of accelerators."
  resolution: "Request a quota increase via Service Quotas or delete unused accelerators."

- symptoms: "BYOIP address not available"
  diagnosis: "BYOIP CIDR is not in ADVERTISING state or is already in use."
  resolution: "Check BYOIP CIDR status with list-byoip-cidrs and ensure it is advertised."

## Output Format

```yaml
root_cause: "creation_failure — <specific_cause>"
evidence:
  - type: iam_permissions
    content: "<caller identity and policy>"
  - type: service_quotas
    content: "<current accelerator count vs limit>"
  - type: cloudtrail
    content: "<error from creation event>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions or request quota increase"
  long_term: "Implement proper IAM policies and monitor quota usage"
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
