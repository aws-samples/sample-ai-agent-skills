---
title: "D3 — App Runner Outbound Connectivity Issues"
description: "Diagnose outbound connectivity failures to external APIs, databases, and services"
status: active
severity: MEDIUM
triggers:
  - "outbound connectivity"
  - "cannot reach external API"
  - "connection timeout"
  - "DNS resolution failed"
  - "network error"
  - "ECONNREFUSED"
owner: devops-agent
objective: "Identify and fix outbound connectivity issues from App Runner services"
context: "App Runner services can reach public internet endpoints by default without a VPC connector. When a VPC connector is attached, outbound traffic routing depends on the destination — public endpoints are still reachable directly, but VPC-routed traffic goes through the VPC subnets. Common issues include DNS resolution failures, firewall rules, VPC routing problems, and external service availability."
---

## Phase 1 — Triage

MUST:
- Check if a VPC connector is attached: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.NetworkConfiguration'`
- Review application logs for connection error details: `aws logs get-log-events --log-group-name /aws/apprunner/<service-name>/<service-id>/application --log-stream-name <stream>`
- Check if the target endpoint is public or private
- If VPC connector attached, check subnet route tables for internet access (NAT gateway): `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify the external service is reachable and not experiencing an outage

SHOULD:
- Check security group outbound rules on the VPC connector: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].IpPermissionsEgress'`
- Verify DNS resolution for the target endpoint
- Check if network ACLs are blocking outbound traffic

MAY:
- Check VPC flow logs for rejected outbound traffic
- Verify the external service's firewall allows traffic from AWS IP ranges
- Test connectivity from an EC2 instance in the same VPC/subnet

## Phase 2 — Remediate

MUST:
- For public endpoints without VPC connector: verify the endpoint URL and DNS are correct
- For VPC-routed traffic: ensure subnets have a NAT gateway for internet-bound traffic
- Ensure security group outbound rules allow traffic to the target port

SHOULD:
- Use VPC endpoints for AWS services to avoid routing through NAT gateway
- Implement connection timeouts and retries in the application
- Use HTTPS for all external connections

MAY:
- Whitelist App Runner's NAT gateway Elastic IP in the external service's firewall
- Implement a health check for external dependencies in the application

## Common Issues

- symptoms: "Application cannot reach external API after attaching VPC connector"
  diagnosis: "VPC connector subnets do not have a NAT gateway for internet-bound traffic."
  resolution: "Add a NAT gateway to the VPC connector subnets, or use public subnets with an internet gateway."

- symptoms: "DNS resolution fails for external hostname"
  diagnosis: "VPC DNS settings are misconfigured or the VPC connector subnets use custom DNS."
  resolution: "Ensure VPC has enableDnsSupport and enableDnsHostnames enabled."

- symptoms: "Connection to external API times out intermittently"
  diagnosis: "Security group or NACL rules are too restrictive, or NAT gateway is throttled."
  resolution: "Check security group outbound rules. Verify NACL allows return traffic. Check NAT gateway metrics."

## Output Format

```yaml
root_cause: "outbound_connectivity — <specific_cause>"
evidence:
  - type: network_config
    content: "<VPC connector, subnets, security groups>"
  - type: application_logs
    content: "<connection error details>"
  - type: route_table
    content: "<routing configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix routing, security groups, or DNS configuration"
  long_term: "Implement connection resilience and monitor outbound connectivity"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Verify endpoint URL and DNS | GREEN | Read-only check |
| Ensure subnets have NAT gateway | YELLOW | Network infrastructure change |
| Ensure security group allows outbound | YELLOW | Security group rule change |
| Use VPC endpoints for AWS services | YELLOW | New VPC endpoint creation |
| Implement connection timeouts/retries | YELLOW | Application code change |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Network configuration**: VPC connector, subnets, security groups — INTERNAL
- **Application logs**: Connection error details — INTERNAL
- **Route tables**: Routing configuration — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER whitelist 0.0.0.0/0 in security group outbound rules without justification

## Phase 3 — Rollback

1. Revert security group outbound rule changes
2. Revert NAT gateway changes if they were modified
3. Revert VPC endpoint creation if it caused issues
4. Verify outbound connectivity is restored to previous state
5. Monitor application logs for connection errors after rollback

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
