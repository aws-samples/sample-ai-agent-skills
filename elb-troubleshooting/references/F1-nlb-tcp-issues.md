---
title: "F1 — NLB TCP Issues"
description: "Diagnose and resolve Network Load Balancer TCP connection and routing issues"
status: active
severity: HIGH
triggers:
  - "NLB connection failed"
  - "TCP connection timeout"
  - "NLB target unreachable"
  - "client IP preservation"
  - "NLB health check"
owner: devops-agent
objective: "Resolve NLB TCP connection issues and restore proper Layer 4 load balancing"
context: "NLB operates at Layer 4 (TCP/UDP/TLS) and provides ultra-low latency and high throughput. NLB preserves the client source IP by default (for instance targets), which requires targets to accept traffic from any IP. NLB uses flow hash routing for connection persistence. Common issues include security group configuration (NLB does not have its own SG), health check failures, client IP preservation conflicts, and idle timeout (fixed 350s for TCP)."
---

## Phase 1 — Triage

MUST:
- Check NLB target health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Verify target security group allows traffic from client IPs (NLB preserves source IP): `aws ec2 describe-security-groups --group-ids <target-sg-id>`
- Check NLB listener configuration: `aws elbv2 describe-listeners --load-balancer-arn <lb-arn>`
- Verify the target is listening on the expected port: `ss -tlnp | grep <port>` (on target)
- Check if client IP preservation is enabled: `aws elbv2 describe-target-group-attributes --target-group-arn <tg-arn> --query 'Attributes[?Key==\`preserve_client_ip.enabled\`]'`

SHOULD:
- Check NLB flow logs (if enabled) for connection details
- Verify the target's health check port and protocol: `aws elbv2 describe-target-groups --target-group-arns <tg-arn>`
- Check if the target can handle the NLB health check (TCP or HTTP)
- Verify NACLs allow traffic on the target port from all source IPs (due to client IP preservation)

MAY:
- Check CloudWatch metrics: NewFlowCount, ActiveFlowCount, ProcessedBytes
- Verify DNS resolution of the NLB endpoint
- Test connectivity directly to the NLB DNS name: `nc -zv <nlb-dns> <port>`

## Phase 2 — Remediate

MUST:
- If security group blocks client IPs: add inbound rule allowing traffic from 0.0.0.0/0 on the target port (required when client IP preservation is on), or disable client IP preservation
- If health check fails: verify the target application responds on the health check port/protocol
- If target not listening: start the application on the correct port
- If NACL blocks traffic: add allow rules for the target port from all source IPs

SHOULD:
- For IP-type targets: client IP preservation is not available — ensure the target accepts traffic from NLB subnet IPs
- Configure TCP keep-alive on targets with interval less than 350 seconds (NLB idle timeout)
- Verify connection draining settings for graceful target deregistration

MAY:
- Enable NLB access logs for connection-level debugging
- Implement proxy protocol v2 if the target needs the original client IP with IP-type targets
- Set up CloudWatch alarms for UnHealthyHostCount and TCP_ELB_Reset_Count

## Common Issues

- symptoms: "Targets healthy but clients cannot connect"
  diagnosis: "Target security group only allows traffic from the VPC CIDR, but NLB preserves client source IP."
  resolution: "Allow inbound traffic from 0.0.0.0/0 on the target port, or disable client IP preservation."

- symptoms: "Connections drop after 350 seconds of inactivity"
  diagnosis: "NLB TCP idle timeout is fixed at 350 seconds and cannot be changed."
  resolution: "Implement TCP keep-alive on both client and target with intervals less than 350 seconds."

- symptoms: "Health checks pass but some connections fail"
  diagnosis: "Health check uses a different port/protocol than the traffic port."
  resolution: "Align health check configuration with the actual service port and protocol."

- symptoms: "NLB returns RST packets to clients"
  diagnosis: "Target is not available or has rejected the connection. Check TCP_ELB_Reset_Count metric."
  resolution: "Verify target application is running and accepting connections. Check for connection limits."

## Output Format

```yaml
root_cause: "NLB TCP issue — <specific_cause>"
evidence:
  - type: target_health
    content: "<target health status>"
  - type: security_groups
    content: "<target security group rules>"
severity: HIGH
mitigation:
  immediate: "Fix security group, health check, or target configuration"
  long_term: "Implement proper security group rules and TCP keep-alive configuration"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get): GREEN — read-only"
  - "Phase 2 security group changes: YELLOW — affects network access"
  - "Phase 2 client IP preservation toggle: YELLOW — changes how targets see source IPs"

## Escalation Conditions

escalation_conditions:
  - "NLB connectivity failure affects production services"
  - "Security group changes to allow 0.0.0.0/0 require security review"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Issue involves NLB static IP or Elastic IP configuration"
  - "Proxy protocol configuration changes affect application behavior"

## Data Sensitivity

data_sensitivity:
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Network access rules and IP ranges"
  - command: "NLB flow logs"
    sensitivity: HIGH
    contains: "Connection details including client and target IPs"
  - command: "describe-target-health"
    sensitivity: LOW
    contains: "Target IDs and health status"

## Prohibited Actions

prohibited_actions:
  - "NEVER disable client IP preservation without understanding the impact on application logging and security"
  - "NEVER open all ports on target security groups to troubleshoot NLB issues"
  - "NEVER modify NLB Elastic IP associations on production load balancers without planning"
  - "NEVER ignore TCP RST metrics — they indicate connection failures"

## Phase 3 — Rollback

- Before changing security groups, document current rules
- If client IP preservation was toggled, revert if it breaks application functionality
- If security group changes are too permissive, tighten them after identifying the required source IPs
- If proxy protocol was enabled, disable it if the target cannot parse the protocol header
- Monitor connection success rates and health check status after any changes
