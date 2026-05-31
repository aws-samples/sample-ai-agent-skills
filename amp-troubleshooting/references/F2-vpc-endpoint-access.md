---
title: "F2 — VPC Endpoint Access"
description: "Diagnose and resolve VPC endpoint connectivity issues for AMP"
status: active
severity: MEDIUM
triggers:
  - "VPC endpoint"
  - "private connectivity"
  - "PrivateLink"
  - "vpce"
  - "private access"
  - "no internet"
owner: devops-agent
objective: "Identify and fix VPC endpoint configuration for private AMP access"
context: "AMP supports VPC endpoints (AWS PrivateLink) via com.amazonaws.<region>.aps-workspaces for private connectivity. Without a VPC endpoint, collectors in a VPC must route through NAT gateway or internet gateway to reach AMP. VPC endpoints keep traffic within the AWS network."
---

## Phase 1 — Triage

MUST:
- Check if VPC endpoint exists: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.aps-workspaces`
- Verify endpoint status is 'available': check `State` in describe output
- Check endpoint security group allows HTTPS (443) inbound from collector subnets
- Verify private DNS is enabled on the endpoint (required for transparent DNS resolution)

SHOULD:
- Check endpoint subnet associations — must include subnets where collectors run
- Verify the VPC endpoint policy allows the required AMP actions
- Test DNS resolution from within the VPC: `nslookup aps-workspaces.<region>.amazonaws.com`
- Check if the collector can reach the endpoint: `curl -v https://aps-workspaces.<region>.amazonaws.com`

MAY:
- Check VPC route tables for any conflicting routes
- Verify no NACL rules block traffic to the endpoint
- Check if multiple VPC endpoints exist (only one per service per VPC)

## Phase 2 — Remediate

MUST:
- If endpoint missing: create it:
  ```
  aws ec2 create-vpc-endpoint --vpc-id <vpc-id> \
    --service-name com.amazonaws.<region>.aps-workspaces \
    --vpc-endpoint-type Interface \
    --subnet-ids <subnet-1> <subnet-2> \
    --security-group-ids <sg-id> \
    --private-dns-enabled
  ```
- If security group blocks traffic: add inbound rule for HTTPS (443) from collector CIDR
- If private DNS not enabled: modify endpoint to enable it
- If endpoint in wrong subnets: add the correct subnet associations

SHOULD:
- Enable private DNS so collectors use the standard endpoint URL without changes
- Scope the endpoint policy to allow only required AMP actions
- Place the endpoint in the same subnets as the collectors for optimal routing

MAY:
- Set up CloudWatch monitoring for VPC endpoint metrics
- Implement endpoint policies for fine-grained access control
- Configure endpoint for cross-AZ resilience

## Common Issues

- symptoms: "Collector timeouts after removing NAT gateway"
  diagnosis: "No VPC endpoint configured — collector cannot reach AMP without internet access."
  resolution: "Create a VPC endpoint for com.amazonaws.<region>.aps-workspaces with private DNS enabled."

- symptoms: "VPC endpoint exists but collector still times out"
  diagnosis: "Endpoint security group doesn't allow inbound HTTPS from collector."
  resolution: "Add inbound rule: port 443, source = collector security group or CIDR."

- symptoms: "DNS resolution returns public IP instead of private endpoint IP"
  diagnosis: "Private DNS not enabled on the VPC endpoint."
  resolution: "Modify endpoint to enable private DNS. Requires VPC DNS support and DNS hostnames enabled."

- symptoms: "Endpoint available but intermittent connectivity"
  diagnosis: "Endpoint not in all AZs where collectors run."
  resolution: "Add subnet associations for all AZs where collector pods/instances are deployed."

## Safety Ratings

```
safety_ratings:
  - "describe-vpc-endpoints: GREEN — read-only endpoint check"
  - "nslookup/DNS resolution test: GREEN — read-only connectivity test"
  - "create-vpc-endpoint: YELLOW — creates network infrastructure with ongoing cost"
  - "modify endpoint security group: YELLOW — changes network access controls"
```

## Escalation Conditions

- VPC endpoint creation requires network team approval
- Security group changes affect other services using the same endpoint
- Private DNS enablement requires VPC DNS settings changes
- Multiple AZs need endpoint coverage for high availability

## Data Sensitivity

- VPC endpoint configuration: MEDIUM — reveals network architecture
- Security group rules: MEDIUM — reveals network access controls
- DNS resolution results: LOW — reveals endpoint IP addresses

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER delete a VPC endpoint that other services depend on
- NEVER disable private DNS without understanding the impact on other services

## Phase 3 — Rollback

1. If a VPC endpoint was created incorrectly, delete it: `aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <vpce-id>`
2. If security group rules were changed, restore original rules
3. If private DNS was enabled/disabled, restore original setting
4. Reconfigure collectors to use NAT gateway or internet gateway if VPC endpoint is removed
5. Verify all collectors can reach the AMP endpoint after rollback

## Output Format

```yaml
root_cause: "VPC endpoint access — <specific_cause>"
evidence:
  - type: vpc_endpoint
    content: "<endpoint status, subnets, security groups>"
  - type: connectivity_test
    content: "<DNS resolution and connectivity results>"
severity: MEDIUM
mitigation:
  immediate: "Fix VPC endpoint configuration or security group rules"
  long_term: "Implement VPC endpoint monitoring, document network architecture"
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
