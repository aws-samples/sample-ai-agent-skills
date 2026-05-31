---
title: "C1 — Endpoint Creation Issues"
description: "Diagnose Verified Access endpoint creation failures"
status: active
severity: HIGH
triggers:
  - "endpoint creation failed"
  - "cannot create endpoint"
  - "endpoint error"
owner: devops-agent
objective: "Identify why endpoint creation is failing and resolve the issue"
context: "Endpoints connect Verified Access to backend applications via load balancers or network interfaces. Endpoints belong to a group and require VPC configuration including subnets and security groups. The backend must be accessible from the endpoint's VPC."
---

## Phase 1 — Triage

MUST:
- Check existing endpoints: `aws ec2 describe-verified-access-endpoints`
- Verify the group exists: `aws ec2 describe-verified-access-groups --verified-access-group-ids <group-id>`
- Check IAM permissions for ec2:CreateVerifiedAccessEndpoint
- Verify VPC, subnet, and security group configuration

SHOULD:
- Check if the backend (ALB/NLB/ENI) exists and is healthy
- Verify endpoint quota limits
- Check CloudTrail for creation errors

MAY:
- Verify the backend is in the same VPC as the endpoint
- Check for conflicting endpoint configurations

## Phase 2 — Remediate

MUST:
- Create load balancer endpoint: `aws ec2 create-verified-access-endpoint --verified-access-group-id <group-id> --endpoint-type load-balancer --attachment-type vpc --load-balancer-options LoadBalancerArn=<alb-arn>,Port=443,Protocol=https,SubnetIds=<subnet-ids> --security-group-ids <sg-ids> --domain-certificate-arn <cert-arn> --endpoint-domain-prefix <prefix>`
- Ensure security groups allow traffic from endpoint to backend

SHOULD:
- Use HTTPS for backend communication
- Configure appropriate security groups

MAY:
- Create network interface endpoint for non-LB backends
- Tag endpoints for management

## Common Issues

- symptoms: "Endpoint creation fails with VPC error"
  diagnosis: "Subnet or security group not in the correct VPC."
  resolution: "Verify all resources are in the same VPC."

- symptoms: "Load balancer not found"
  diagnosis: "ALB/NLB ARN incorrect or in different region."
  resolution: "Verify load balancer ARN and region."

- symptoms: "Endpoint stuck in pending"
  diagnosis: "Backend not reachable or security group blocking."
  resolution: "Check security groups allow endpoint to reach backend on specified port."

## Output Format

```yaml
root_cause: "endpoint_creation — <specific_cause>"
evidence:
  - type: endpoint_config
    content: "<endpoint configuration>"
  - type: backend_status
    content: "<backend health>"
  - type: vpc_config
    content: "<VPC, subnet, security group>"
severity: HIGH
mitigation:
  immediate: "Fix VPC/security group configuration"
  long_term: "Use IaC for endpoint deployment"
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

- Trust provider configurations contain identity provider details
- Access logs reveal user access patterns and device information
- Cedar policies define authorization logic for applications
- Classification: CONFIDENTIAL - access control policies and user activity data

## Prohibited Actions

- NEVER modify trust provider configurations during active user sessions
- NEVER change Verified Access policies that could lock out production users
- NEVER delete endpoints serving active application traffic
- NEVER modify Cedar policies without testing in a non-production environment first

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
