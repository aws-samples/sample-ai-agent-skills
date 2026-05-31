---
title: "E2 — Global Accelerator Custom Routing Issues"
description: "Diagnose custom routing accelerator configuration and port mapping problems"
status: active
severity: MEDIUM
triggers:
  - "custom routing"
  - "port mapping"
  - "deterministic routing"
  - "custom routing accelerator"
owner: devops-agent
objective: "Resolve custom routing accelerator configuration and port mapping issues"
context: "Custom routing accelerators map specific ports to specific EC2 instances in VPC subnets. They are separate from standard accelerators and only support EC2 instance endpoints via subnets."
---

## Phase 1 — Triage

MUST:
- List custom routing accelerators: `aws globalaccelerator list-custom-routing-accelerators`
- Describe custom routing accelerator: `aws globalaccelerator describe-custom-routing-accelerator --accelerator-arn <accelerator-arn>`
- List custom routing listeners: `aws globalaccelerator list-custom-routing-listeners --accelerator-arn <accelerator-arn>`
- List port mappings: `aws globalaccelerator list-custom-routing-port-mappings --accelerator-arn <accelerator-arn>`
- Check endpoint group: `aws globalaccelerator list-custom-routing-endpoint-groups --listener-arn <listener-arn>`

SHOULD:
- Verify the VPC subnet has EC2 instances running
- Check if traffic is allowed to specific destinations: `aws globalaccelerator list-custom-routing-port-mappings-by-destination --endpoint-id <subnet-id> --destination-address <instance-private-ip>`
- Verify security groups on EC2 instances allow traffic on mapped ports

MAY:
- Check if custom routing destinations are allowed or denied
- Review CloudTrail for custom routing configuration changes

## Phase 2 — Remediate

MUST:
- Ensure custom routing endpoint group references valid VPC subnets
- Allow traffic to specific destinations: `aws globalaccelerator allow-custom-routing-traffic --endpoint-group-arn <endpoint-group-arn> --endpoint-id <subnet-id> --destination-addresses <ip> --destination-ports <port>`
- Verify EC2 instances are running in the specified subnets

SHOULD:
- Configure appropriate port ranges on the listener
- Ensure subnet has sufficient IP addresses for the port mapping

MAY:
- Deny traffic to specific destinations if needed: `aws globalaccelerator deny-custom-routing-traffic --endpoint-group-arn <endpoint-group-arn> --endpoint-id <subnet-id> --destination-addresses <ip>`

## Common Issues

- symptoms: "Custom routing port mapping returns empty"
  diagnosis: "No EC2 instances in the specified subnet or subnet not added to endpoint group."
  resolution: "Ensure EC2 instances are running in the subnet and subnet is added to the endpoint group."

- symptoms: "Traffic not reaching EC2 instance through custom routing"
  diagnosis: "Traffic to the destination is denied (default is deny all)."
  resolution: "Use allow-custom-routing-traffic to permit traffic to specific destinations."

- symptoms: "Cannot use ALB/NLB with custom routing"
  diagnosis: "Custom routing only supports EC2 instances via VPC subnets."
  resolution: "Use a standard accelerator for ALB/NLB endpoints."

## Output Format

```yaml
root_cause: "custom_routing — <specific_cause>"
evidence:
  - type: custom_routing_config
    content: "<custom routing accelerator configuration>"
  - type: port_mappings
    content: "<port mapping details>"
  - type: traffic_permissions
    content: "<allow/deny status for destinations>"
severity: MEDIUM
mitigation:
  immediate: "Allow traffic to destinations or fix subnet configuration"
  long_term: "Document port mapping scheme and monitor custom routing health"
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
