---
title: "E2 — SSM Agent Connectivity"
description: "Diagnose SSM Agent connectivity issues preventing Fleet Manager communication"
status: active
severity: HIGH
triggers: ["SSM Agent offline", "connection lost", "agent not connecting", "ping status inactive"]
owner: devops-agent
objective: "Resolve SSM Agent connectivity issues to restore Fleet Manager functionality"
context: "SSM Agent requires outbound HTTPS (443) connectivity to SSM service endpoints. In private subnets, VPC endpoints or NAT gateways are required. Agent communicates via ssm, ssmmessages, and ec2messages endpoints."
---

## Phase 1 — Triage

MUST:
- Check agent status: `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<instance-id> --query 'InstanceInformationList[*].{Id:InstanceId,Ping:PingStatus,LastPing:LastPingDateTime,Agent:AgentVersion}'`
- Check real-time connection: `aws ssm get-connection-status --target <instance-id>`
- Verify VPC endpoints exist if in private subnet: `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id> --query 'VpcEndpoints[*].{Service:ServiceName,State:State}'`

SHOULD:
- Check security group outbound rules allow HTTPS (443)
- Verify route table has path to SSM endpoints (NAT gateway or VPC endpoint)

MAY:
- Check SSM Agent logs on the instance for connectivity errors
- Test DNS resolution for SSM endpoints from the instance

## Phase 2 — Remediate

MUST:
- Ensure outbound HTTPS (443) is allowed in security groups
- For private subnets, create VPC endpoints: `aws ec2 create-vpc-endpoint --vpc-id <vpc-id> --service-name com.amazonaws.<region>.ssm --vpc-endpoint-type Interface --subnet-ids <subnet-id> --security-group-ids <sg-id>`
- Create all three required endpoints: ssm, ssmmessages, ec2messages

SHOULD:
- Verify VPC endpoint security groups allow inbound HTTPS from instance security groups
- Check NACLs are not blocking HTTPS traffic

MAY:
- Configure SSM Agent proxy settings if using a proxy server
- Set up CloudWatch alarms for SSM Agent connectivity monitoring

## Common Issues

- symptoms: "PingStatus shows 'ConnectionLost' for instances in private subnet"
  diagnosis: "Missing VPC endpoints or NAT gateway for SSM connectivity."
  resolution: "Create VPC endpoints for ssm, ssmmessages, and ec2messages services."

- symptoms: "Agent was connected but lost connectivity after VPC changes"
  diagnosis: "Route table or security group changes broke SSM endpoint access."
  resolution: "Review recent VPC changes. Verify route tables and security groups allow HTTPS to SSM endpoints."

## Safety Ratings

```
safety_ratings:
  - "describe-instance-information: GREEN — read-only agent status"
  - "get-connection-status: GREEN — read-only connectivity check"
  - "describe-vpc-endpoints: GREEN — read-only VPC endpoint listing"
  - "create-vpc-endpoint: YELLOW — creates VPC endpoint resource with cost implications"
```

## Escalation Conditions

- All instances in a VPC lose SSM connectivity simultaneously
- VPC endpoint creation fails due to service limits or VPC configuration
- Network team cannot modify security groups or route tables
- SSM Agent connectivity requires proxy configuration that conflicts with other applications

## Data Sensitivity

- `aws ssm describe-instance-information`: LOW — instance metadata and agent status
- `aws ec2 describe-vpc-endpoints`: MEDIUM — reveals VPC network architecture
- `aws ec2 describe-security-groups`: MEDIUM — reveals network security configuration
- SSM Agent logs: MEDIUM — may contain endpoint URLs and error details

## Prohibited Actions

- NEVER open all outbound ports (0.0.0.0/0 all protocols) to fix connectivity
- NEVER modify VPC route tables without understanding the full network topology
- NEVER disable NACLs to troubleshoot connectivity
- NEVER create VPC endpoints in the wrong subnets or with overly permissive security groups

## Phase 3 — Rollback

1. If VPC endpoints were created incorrectly, delete them: `aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <endpoint-id>`
2. If security group rules were added, remove them: `aws ec2 revoke-security-group-egress --group-id <sg-id> --protocol tcp --port 443 --cidr <cidr>`
3. If route table entries were modified, revert to previous configuration
4. Verify SSM Agent connectivity is restored after changes
5. Document all network changes for the infrastructure team

## Output Format

```yaml
root_cause: "ssm_agent_connectivity — <specific_cause>"
evidence:
  - type: connectivity_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore SSM Agent connectivity"
  long_term: "Implement VPC endpoints and monitoring"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
