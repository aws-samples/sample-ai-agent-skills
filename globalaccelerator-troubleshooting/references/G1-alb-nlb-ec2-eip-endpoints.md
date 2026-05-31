---
title: "G1 — Global Accelerator ALB/NLB/EC2/EIP Endpoint Integration"
description: "Diagnose endpoint integration issues with ALB, NLB, EC2, and EIP"
status: active
severity: HIGH
triggers:
  - "ALB endpoint"
  - "NLB endpoint"
  - "EC2 endpoint"
  - "EIP endpoint"
  - "endpoint integration"
owner: devops-agent
objective: "Resolve integration issues between Global Accelerator and ALB/NLB/EC2/EIP endpoints"
context: "Global Accelerator supports ALB, NLB, EC2 instances, and Elastic IP addresses as endpoints. Each type has specific requirements and behaviors."
---

## Phase 1 — Triage

MUST:
- Check endpoint configuration and health: `aws globalaccelerator describe-endpoint-group --endpoint-group-arn <endpoint-group-arn> --query 'EndpointGroup.EndpointDescriptions'`
- Verify the endpoint resource exists:
  - ALB: `aws elbv2 describe-load-balancers --load-balancer-arns <alb-arn>`
  - NLB: `aws elbv2 describe-load-balancers --load-balancer-arns <nlb-arn>`
  - EC2: `aws ec2 describe-instances --instance-ids <instance-id>`
  - EIP: `aws ec2 describe-addresses --allocation-ids <allocation-id>`
- Verify endpoint is in the same region as the endpoint group
- Check endpoint security groups (ALB/EC2): `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- For ALB/NLB: verify target group health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- For EC2: verify instance is running and has a public IP
- For EIP: verify EIP is associated with a running instance
- Check if client IP preservation is enabled (ALB only)

MAY:
- Test direct connectivity to the endpoint bypassing GA
- Check VPC routing tables for the endpoint subnet
- Verify NACLs allow traffic

## Phase 2 — Remediate

MUST:
- Ensure endpoint resource exists, is running, and is in the correct region
- Fix security group rules to allow traffic from GA health checks and client traffic
- For EC2 endpoints, ensure the instance has a public IP or is behind an ALB/NLB

SHOULD:
- Enable client IP preservation for ALB endpoints if needed
- Ensure ALB/NLB listeners match the GA listener port configuration

MAY:
- Consider using NLB for TCP/UDP workloads and ALB for HTTP/HTTPS
- Set up cross-zone load balancing on ALB/NLB

## Common Issues

- symptoms: "ALB endpoint unhealthy in GA but ALB itself is healthy"
  diagnosis: "GA health check port/path differs from ALB health check."
  resolution: "Align GA health check configuration with ALB listener/health check settings."

- symptoms: "EC2 endpoint not receiving traffic"
  diagnosis: "EC2 instance does not have a public IP address."
  resolution: "Assign a public IP or Elastic IP to the EC2 instance, or use ALB/NLB instead."

- symptoms: "Client IP not preserved at ALB"
  diagnosis: "Client IP preservation is not enabled for the ALB endpoint."
  resolution: "Enable client IP preservation on the endpoint configuration."

- symptoms: "NLB endpoint connection timeout"
  diagnosis: "NLB security group or target group health check is misconfigured."
  resolution: "Verify NLB target group health and security group rules."

## Output Format

```yaml
root_cause: "endpoint_integration — <specific_cause>"
evidence:
  - type: endpoint_config
    content: "<endpoint type and configuration>"
  - type: endpoint_health
    content: "<GA and ELB health status>"
  - type: security_groups
    content: "<security group rules>"
severity: HIGH
mitigation:
  immediate: "Fix endpoint configuration or security groups"
  long_term: "Align GA and endpoint health checks, implement monitoring"
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
