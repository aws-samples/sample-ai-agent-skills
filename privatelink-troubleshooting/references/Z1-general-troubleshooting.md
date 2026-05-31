---
title: "Z1 — General PrivateLink Troubleshooting"
description: "Catch-all systematic investigation for PrivateLink issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "PrivateLink issue"
  - "VPC endpoint error"
  - "PrivateLink problem"
  - "endpoint not working"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate PrivateLink issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for PrivateLink issues that don't clearly match a specific runbook. It covers endpoint health checks, DNS verification, security group analysis, connectivity testing, and systematic elimination of common causes. Use this as a starting point when the issue category is unclear, then pivot to a specific runbook once the problem area is identified."
---

## Phase 1 — Triage

MUST:
- Check all endpoints in the VPC: `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id> --query 'VpcEndpoints[*].{Id:VpcEndpointId,Service:ServiceName,State:State,Type:VpcEndpointType,PrivateDns:PrivateDnsEnabled}'`
- Check endpoint ENIs and security groups: `aws ec2 describe-network-interfaces --filters Name=vpc-endpoint-id,Values=<endpoint-id> --query 'NetworkInterfaces[*].{Id:NetworkInterfaceId,SubnetId:SubnetId,AZ:AvailabilityZone,PrivateIp:PrivateIpAddress,Groups:Groups}'`
- Check VPC DNS attributes:
  - `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`
  - `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsHostnames`
- Check endpoint DNS entries: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id> --query 'VpcEndpoints[*].DnsEntries'`
- Check security group rules on endpoint ENIs: `aws ec2 describe-security-groups --group-ids <sg-ids>`

SHOULD:
- Check CloudTrail for recent endpoint events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com --max-results 20`
- Check endpoint policy: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id> --query 'VpcEndpoints[*].PolicyDocument'`
- For endpoint services, check connection status: `aws ec2 describe-vpc-endpoint-connections --filters Name=vpc-endpoint-id,Values=<endpoint-id>`
- Check VPC Flow Logs for traffic through endpoint ENIs

MAY:
- Check AWS Health Dashboard for service-level issues
- Review application logs for connection errors
- Test connectivity from within the VPC to the endpoint ENI private IP

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Endpoint issues → A1 (creation), A2 (DNS), A3 (status)
  - Service issues → B1 (service creation), B2 (NLB), B3 (acceptance)
  - Connectivity → C1 (connection failures), C2 (security groups), C3 (cross-AZ)
  - DNS → D1 (private DNS), D2 (split-horizon)
  - Cross-account → E1 (cross-account access), E2 (endpoint policies)
  - Performance → F1 (throughput), F2 (latency)
  - Security → G1 (endpoint policies), G2 (security groups audit)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues (e.g., DNS + security group)
- Review recent changes that may have caused the issue
- Verify the fix resolves the issue without introducing new problems

MAY:
- Set up comprehensive monitoring to prevent recurrence
- Create a post-incident report for significant issues

## Common Issues

- symptoms: "Cannot connect to AWS service through endpoint"
  diagnosis: "Multiple possible causes: DNS, security groups, endpoint state, endpoint policy."
  resolution: "Follow triage steps. Check endpoint state → DNS → security groups → endpoint policy."

- symptoms: "Endpoint was working but stopped"
  diagnosis: "Recent change to security groups, endpoint policy, or VPC configuration."
  resolution: "Check CloudTrail for recent changes. Revert if a change caused the issue."

- symptoms: "Some instances can connect but others cannot"
  diagnosis: "Security group, subnet, or AZ differences between instances."
  resolution: "Compare security groups and subnets of working vs non-working instances."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: endpoint_status
    content: "<endpoint state and configuration>"
  - type: dns_resolution
    content: "<DNS entries and resolution results>"
  - type: security_groups
    content: "<security group rules on endpoint ENIs>"
  - type: connectivity
    content: "<connection test results>"
severity: MEDIUM
general_analysis:
  endpoint_state: "available | pending | failed | rejected"
  private_dns: "enabled | disabled"
  security_groups: "properly configured | misconfigured"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive monitoring and alerting"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — general troubleshooting uses read-only operations
- Service Disruption Risk: LOW — diagnostic commands do not affect endpoints
- Compliance Impact: LOW — standard troubleshooting procedures

## Escalation Conditions
- Issue cannot be resolved using standard troubleshooting procedures
- PrivateLink service becomes unavailable
- Multiple unrelated issues occur simultaneously
- AWS Health Dashboard shows VPC service event
- Issue requires AWS Support engagement

## Data Sensitivity
- Endpoint configurations — INTERNAL
- Network diagnostics — INTERNAL
- CloudWatch metrics — INTERNAL
- Error messages — INTERNAL
- Never expose VPC architecture details externally

## Prohibited Actions
- Do NOT make changes to production endpoints without approval
- Do NOT run destructive commands during initial diagnosis
- Do NOT share endpoint details in public forums without sanitizing
- Do NOT bypass established escalation procedures
- Do NOT attempt fixes outside PrivateLink service boundaries
- Do NOT ignore recurring issues

## Phase 3 — Rollback
- Revert any diagnostic changes made during troubleshooting
- Restore endpoints to last known good configuration
- Document all troubleshooting steps and outcomes
- Verify endpoint health after troubleshooting
- Update runbooks with new findings
- Close out temporary access granted for troubleshooting

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
