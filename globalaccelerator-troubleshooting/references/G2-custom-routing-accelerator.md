---
title: "G2 — Global Accelerator Custom Routing Accelerator Integration"
description: "Diagnose custom routing accelerator setup and integration issues"
status: active
severity: MEDIUM
triggers:
  - "custom routing accelerator"
  - "custom routing setup"
  - "custom routing integration"
owner: devops-agent
objective: "Resolve custom routing accelerator setup and integration problems"
context: "Custom routing accelerators are a separate resource type from standard accelerators. They map port ranges to EC2 instances in VPC subnets for deterministic routing."
---

## Phase 1 — Triage

MUST:
- Describe custom routing accelerator: `aws globalaccelerator describe-custom-routing-accelerator --accelerator-arn <accelerator-arn>`
- List custom routing listeners: `aws globalaccelerator list-custom-routing-listeners --accelerator-arn <accelerator-arn>`
- List custom routing endpoint groups: `aws globalaccelerator list-custom-routing-endpoint-groups --listener-arn <listener-arn>`
- Check port mappings: `aws globalaccelerator list-custom-routing-port-mappings --accelerator-arn <accelerator-arn> --max-results 100`

SHOULD:
- Verify VPC subnet has running EC2 instances: `aws ec2 describe-instances --filters Name=subnet-id,Values=<subnet-id> Name=instance-state-name,Values=running`
- Check traffic allow/deny status for destinations
- Verify the subnet CIDR has enough IPs for port mapping

MAY:
- Check custom routing accelerator attributes: `aws globalaccelerator describe-custom-routing-accelerator-attributes --accelerator-arn <accelerator-arn>`
- Review the port range configuration on the listener

## Phase 2 — Remediate

MUST:
- Ensure custom routing endpoint group references valid VPC subnets with running EC2 instances
- Allow traffic to required destinations: `aws globalaccelerator allow-custom-routing-traffic --endpoint-group-arn <endpoint-group-arn> --endpoint-id <subnet-id> --destination-addresses <ip> --destination-ports <port>`
- Configure listener with appropriate port ranges

SHOULD:
- Document the port-to-instance mapping for operational reference
- Ensure EC2 instances have security groups allowing traffic on mapped ports

MAY:
- Use allow-all-traffic for testing: `aws globalaccelerator allow-custom-routing-traffic --endpoint-group-arn <endpoint-group-arn> --endpoint-id <subnet-id> --allow-all-traffic-to-endpoint`

## Common Issues

- symptoms: "Custom routing accelerator created but no port mappings"
  diagnosis: "No endpoint groups or subnets added to the custom routing accelerator."
  resolution: "Create endpoint groups and add VPC subnets with EC2 instances."

- symptoms: "Port mapping exists but traffic not reaching instance"
  diagnosis: "Traffic to the destination is denied (default deny) or security group blocks it."
  resolution: "Allow traffic to the destination and update EC2 security groups."

- symptoms: "Trying to add ALB to custom routing accelerator"
  diagnosis: "Custom routing only supports EC2 instances via VPC subnets."
  resolution: "Use a standard accelerator for ALB/NLB endpoints."

## Output Format

```yaml
root_cause: "custom_routing_integration — <specific_cause>"
evidence:
  - type: custom_routing_config
    content: "<accelerator and listener configuration>"
  - type: port_mappings
    content: "<port mapping details>"
  - type: ec2_instances
    content: "<EC2 instances in subnet>"
severity: MEDIUM
mitigation:
  immediate: "Fix subnet configuration or allow traffic to destinations"
  long_term: "Document port mapping scheme and automate configuration"
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
