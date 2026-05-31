---
title: "B2 — Global Accelerator Endpoint Group Configuration"
description: "Diagnose endpoint group misconfiguration issues"
status: active
severity: MEDIUM
triggers:
  - "endpoint group error"
  - "endpoint group configuration"
  - "regional endpoint issues"
owner: devops-agent
objective: "Resolve endpoint group configuration problems including region mapping and endpoint limits"
context: "Each endpoint group maps to a specific AWS region and contains up to 10 endpoints. Endpoint groups define health check settings, traffic dial, and endpoint configurations."
---

## Phase 1 — Triage

MUST:
- List all endpoint groups for the listener: `aws globalaccelerator list-endpoint-groups --listener-arn <listener-arn>`
- Describe the endpoint group: `aws globalaccelerator describe-endpoint-group --endpoint-group-arn <endpoint-group-arn>`
- Check endpoint configurations within the group: `aws globalaccelerator describe-endpoint-group --endpoint-group-arn <endpoint-group-arn> --query 'EndpointGroup.EndpointDescriptions'`
- Verify the endpoint group region matches endpoint locations

SHOULD:
- Check if endpoints are valid (ALB/NLB/EC2/EIP exist and are in the correct region)
- Verify endpoint weights are set correctly (0-255)
- Check for duplicate endpoint groups in the same region

MAY:
- Review CloudTrail for recent endpoint group changes
- Check if endpoint group was recently created (propagation time)

## Phase 2 — Remediate

MUST:
- Ensure all endpoints in the group are in the same region as the endpoint group
- Verify endpoint ARNs/IDs are valid and the resources exist
- Fix health check configuration if endpoints are unhealthy

SHOULD:
- Set appropriate endpoint weights for traffic distribution
- Remove stale or deleted endpoints from the group

MAY:
- Create additional endpoint groups in other regions for multi-region failover

## Common Issues

- symptoms: "Endpoint group creation fails with InvalidArgumentException"
  diagnosis: "Endpoint resource is in a different region than the endpoint group."
  resolution: "Ensure all endpoints are in the same region as the endpoint group."

- symptoms: "Traffic not reaching some endpoints in the group"
  diagnosis: "Endpoint weight is set to 0 or endpoint is unhealthy."
  resolution: "Set endpoint weight > 0 and verify endpoint health."

- symptoms: "Cannot add more endpoints to the group"
  diagnosis: "Endpoint group has reached the 10-endpoint limit."
  resolution: "Remove unused endpoints or create additional endpoint groups."

## Output Format

```yaml
root_cause: "endpoint_group — <specific_cause>"
evidence:
  - type: endpoint_group_config
    content: "<endpoint group configuration>"
  - type: endpoints
    content: "<endpoint list with health and weights>"
severity: MEDIUM
mitigation:
  immediate: "Fix endpoint group configuration"
  long_term: "Implement endpoint group monitoring and validation"
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
