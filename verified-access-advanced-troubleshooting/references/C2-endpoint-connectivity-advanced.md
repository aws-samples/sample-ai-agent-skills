---
title: "C2 — Endpoint Connectivity Advanced"
description: "Diagnose advanced endpoint connectivity issues in Verified Access"
status: active
severity: HIGH
triggers: ["endpoint not reachable", "connection timeout through verified access", "endpoint status error", "DNS not resolving for endpoint"]
owner: devops-agent
objective: "Resolve advanced connectivity issues between Verified Access endpoints and backend applications"
context: "Endpoints connect Verified Access to backend applications via ALB, NLB, network interfaces, or RDS. Connectivity requires correct DNS, security groups, subnets, and TLS configuration."
---

## Phase 1 — Triage

MUST:
- Check endpoint status: `aws ec2 describe-verified-access-endpoints --query 'VerifiedAccessEndpoints[*].{Id:VerifiedAccessEndpointId,Type:EndpointType,Status:Status,DomainCertArn:DomainCertificateArn}'`
- Verify security groups: `aws ec2 describe-security-groups --group-ids <endpoint-sg-id> --query 'SecurityGroups[*].{Id:GroupId,IngressRules:IpPermissions}'`
- Check DNS configuration for the endpoint domain

SHOULD:
- Verify the backend application (ALB/NLB/ENI) is healthy and reachable
- Check TLS certificate validity and domain matching
- Review VPC routing between the endpoint and backend

MAY:
- Test direct connectivity to the backend bypassing Verified Access
- Check VPC flow logs for blocked traffic

## Phase 2 — Remediate

MUST:
- Ensure security groups allow traffic from Verified Access to the backend
- Verify DNS CNAME points to the Verified Access endpoint domain
- Confirm TLS certificate covers the application domain

SHOULD:
- Verify backend health checks are passing (ALB/NLB target health)
- Ensure subnets have sufficient IP addresses for ENI creation

MAY:
- Implement health monitoring for endpoint connectivity
- Configure redundant backend targets for high availability

## Common Issues

- symptoms: "Endpoint shows active status but users get connection timeout"
  diagnosis: "Security group does not allow traffic from Verified Access ENIs to the backend."
  resolution: "Update security group to allow inbound traffic from the Verified Access endpoint security group or CIDR range."

## Safety Ratings

```
safety_ratings:
  - "describe-verified-access-endpoints: GREEN — read-only endpoint check"
  - "describe-security-groups: GREEN — read-only security group review"
  - "Security group modifications: YELLOW — changes network access controls"
  - "DNS changes: YELLOW — affects traffic routing for all users"
```

## Escalation Conditions

- Endpoint in persistent error state despite correct configuration
- TLS certificate issues requiring certificate reissuance
- VPC networking issues affecting multiple endpoints
- Backend application failures causing endpoint health degradation

## Data Sensitivity

- `aws ec2 describe-verified-access-endpoints`: MEDIUM — reveals application topology
- Security group rules: MEDIUM — reveals network access controls
- DNS configuration: LOW — reveals domain routing
- VPC flow logs: HIGH — contain source/destination IPs and traffic patterns

## Prohibited Actions

- NEVER modify security groups to allow all traffic (0.0.0.0/0) as a troubleshooting step
- NEVER change DNS records without a rollback plan and TTL consideration
- NEVER delete endpoints to recreate them without migrating DNS first
- NEVER bypass TLS verification as a permanent solution

## Phase 3 — Rollback

1. Revert security group changes to previous rules
2. Restore DNS records to previous values (account for TTL propagation)
3. If endpoint was recreated, update DNS to point to the new endpoint domain
4. Verify connectivity is restored for all users
5. Document the connectivity issue and resolution

## Output Format

```yaml
root_cause: "endpoint_connectivity — <specific_cause>"
evidence:
  - type: connectivity_details
    content: "<endpoint status, security groups, DNS>"
severity: HIGH
mitigation:
  immediate: "Restore endpoint connectivity"
  long_term: "Implement connectivity monitoring and health checks"
```

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
