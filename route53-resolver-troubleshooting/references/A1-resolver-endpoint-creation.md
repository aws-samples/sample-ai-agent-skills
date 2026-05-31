---
title: "A1 — Resolver Endpoint Creation"
description: "Diagnose resolver endpoint creation failures"
status: active
severity: HIGH
triggers: ["cannot create endpoint", "endpoint creation failed", "resolver endpoint error"]
owner: devops-agent
objective: "Resolve resolver endpoint creation issues"
context: "Resolver endpoints require at least 2 IP addresses in different AZs, appropriate security groups, and sufficient subnet capacity. Each IP creates an ENI."
---

## Phase 1 — Triage

MUST:
- Check existing endpoints: `aws route53resolver list-resolver-endpoints --query 'ResolverEndpoints[*].{Id:Id,Name:Name,Direction:Direction,Status:Status}'`
- Verify subnet availability: `aws ec2 describe-subnets --subnet-ids <subnet-id-1> <subnet-id-2> --query 'Subnets[*].{Id:SubnetId,AZ:AvailabilityZone,AvailableIPs:AvailableIpAddressCount}'`
- Check ENI limits: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-DF5E4CA3`

SHOULD:
- Verify the security group allows DNS traffic (TCP/UDP 53)
- Confirm subnets are in different AZs

MAY:
- Review CloudTrail for CreateResolverEndpoint errors
- Check if the VPC has DNS resolution and DNS hostnames enabled

## Phase 2 — Remediate

MUST:
- Ensure at least 2 subnets in different AZs with available IP addresses
- Create security group allowing DNS traffic: inbound TCP/UDP 53 for inbound endpoints, outbound TCP/UDP 53 for outbound endpoints
- Create endpoint: `aws route53resolver create-resolver-endpoint --creator-request-id <id> --name <name> --security-group-ids <sg-id> --direction INBOUND --ip-addresses SubnetId=<subnet-1> SubnetId=<subnet-2>`

SHOULD:
- Enable DNS resolution and DNS hostnames on the VPC
- Use private subnets for resolver endpoints

MAY:
- Create endpoints via CloudFormation for reproducibility
- Tag endpoints for cost allocation and management

## Common Issues

- symptoms: "CreateResolverEndpoint fails with 'insufficient IP addresses'"
  diagnosis: "Subnet does not have enough available IP addresses for the ENI."
  resolution: "Use a subnet with more available IPs or create a new subnet."

- symptoms: "Endpoint created but status stuck in CREATING"
  diagnosis: "ENI creation or security group attachment issue."
  resolution: "Check ENI limits and security group configuration. Delete and recreate if stuck."

## Safety Ratings

```
safety_ratings:
  - "list-resolver-endpoints: GREEN — read-only endpoint listing"
  - "describe-subnets: GREEN — read-only subnet information"
  - "create-resolver-endpoint: YELLOW — creates ENIs and consumes IP addresses"
  - "delete-resolver-endpoint: RED — removes endpoint and all associated ENIs"
```

## Escalation Conditions

- Endpoint creation fails with internal service errors after all prerequisites are met
- ENI limit reached and quota increase is needed
- Endpoint creation requires subnets in AZs that don't exist in the region
- Security requirements mandate specific network configurations for resolver endpoints

## Data Sensitivity

- `aws route53resolver list-resolver-endpoints`: LOW — endpoint metadata only
- `aws ec2 describe-subnets`: MEDIUM — reveals VPC network architecture
- `aws route53resolver create-resolver-endpoint`: MEDIUM — creates network resources
- Endpoint IP addresses: MEDIUM — used by on-premises DNS servers for forwarding

## Prohibited Actions

- NEVER create endpoints with a single IP address in one AZ
- NEVER use public subnets for resolver endpoints without security review
- NEVER assign overly permissive security groups (0.0.0.0/0 all ports) to endpoints
- NEVER delete existing endpoints to recreate them without updating all DNS references

## Phase 3 — Rollback

1. If endpoint was created incorrectly, delete it: `aws route53resolver delete-resolver-endpoint --resolver-endpoint-id <id>`
2. If security groups were created, delete them after removing from endpoint
3. If subnets were created for the endpoint, evaluate if they should be retained
4. Verify no forwarding rules or on-premises DNS servers reference the deleted endpoint IPs
5. Document the rollback and notify the network team

## Output Format

```yaml
root_cause: "resolver_endpoint_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix endpoint creation prerequisites"
  long_term: "Deploy endpoints via IaC with proper subnet planning"
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
  - "NEVER suggest disabling DNSSEC without understanding impact"
  - "NEVER suggest wildcard DNS records without review"
  - "NEVER modify hosted zones without backup"
