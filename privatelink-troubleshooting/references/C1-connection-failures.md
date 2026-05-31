---
title: "C1 — Connection Failures"
description: "Diagnose connectivity failures through VPC interface endpoints"
status: active
severity: HIGH
triggers:
  - "connection failed"
  - "cannot connect through endpoint"
  - "timeout through PrivateLink"
  - "connection refused"
  - "endpoint not reachable"
owner: devops-agent
objective: "Identify and resolve connectivity failures through interface endpoints"
context: "Connectivity through interface endpoints can fail due to endpoint not in available state, security group misconfiguration, DNS resolution issues, endpoint policy restrictions, NLB target health (for endpoint services), or network ACL rules. Traffic flows: client → endpoint ENI → service."
---

## Phase 1 — Triage

MUST:
- Verify endpoint state is available: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id> --query 'VpcEndpoints[*].State'`
- Check endpoint ENI security groups: `aws ec2 describe-network-interfaces --filters Name=vpc-endpoint-id,Values=<endpoint-id> --query 'NetworkInterfaces[*].{Id:NetworkInterfaceId,PrivateIp:PrivateIpAddress,Groups:Groups}'`
- Verify security group allows inbound on service port: `aws ec2 describe-security-groups --group-ids <endpoint-sg-id>`
- Check DNS resolution from the client: verify the service DNS resolves to endpoint private IPs

SHOULD:
- Check client security group allows outbound to endpoint: `aws ec2 describe-security-groups --group-ids <client-sg-id>`
- Check NACLs on both client and endpoint subnets
- Verify endpoint policy allows the action: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id> --query 'VpcEndpoints[*].PolicyDocument'`
- Check VPC Flow Logs for rejected traffic

MAY:
- Test connectivity with telnet/nc to the endpoint ENI private IP on the service port
- Check for route table issues between client subnet and endpoint subnet

## Phase 2 — Remediate

MUST:
- Fix security group rules to allow traffic on the service port (typically 443)
- Ensure endpoint is in available state
- Fix DNS resolution if service DNS doesn't resolve to private IPs

SHOULD:
- Update endpoint policy if it's blocking legitimate traffic
- Fix NACL rules if they're blocking traffic
- Verify both inbound (endpoint) and outbound (client) security group rules

MAY:
- Enable VPC Flow Logs for ongoing traffic analysis
- Set up connectivity monitoring

## Common Issues

- symptoms: "Connection timeout to AWS service through endpoint"
  diagnosis: "Security group on endpoint ENI not allowing inbound HTTPS (443)."
  resolution: "Add inbound rule for port 443 from client CIDR/security group to endpoint security group."

- symptoms: "Connection works from some instances but not others"
  diagnosis: "Client security group or subnet NACL differences."
  resolution: "Check security groups and NACLs for all affected instances. Ensure consistent rules."

- symptoms: "Endpoint policy blocking access"
  diagnosis: "Restrictive endpoint policy not allowing the specific action or principal."
  resolution: "Update endpoint policy to allow the required actions and principals."

## Output Format

```yaml
root_cause: "connection_failure — <specific_cause>"
evidence:
  - type: endpoint_state
    content: "<endpoint availability status>"
  - type: security_groups
    content: "<endpoint and client security group rules>"
  - type: dns_resolution
    content: "<DNS resolution results>"
  - type: flow_logs
    content: "<VPC Flow Log entries if available>"
severity: HIGH
mitigation:
  immediate: "Fix security groups, DNS, or endpoint policy"
  long_term: "Implement connectivity monitoring and automated security group management"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — connection failures prevent data access
- Service Disruption Risk: HIGH — connection failures directly impact applications
- Compliance Impact: MEDIUM — connection issues affect private connectivity requirements

## Escalation Conditions
- All connections through endpoint fail
- Connection failures persist after security group verification
- Connection timeouts exceed acceptable thresholds
- Multiple endpoints experience connection failures simultaneously
- Connection failures correlate with network changes

## Data Sensitivity
- Connection details — INTERNAL
- Security group configurations — INTERNAL
- Network diagnostics — INTERNAL
- Endpoint health data — INTERNAL
- Never expose connection details externally

## Prohibited Actions
- Do NOT bypass security controls to troubleshoot connections
- Do NOT open endpoints to all traffic for testing
- Do NOT ignore persistent connection failures
- Do NOT modify endpoint configurations during active incidents without approval
- Do NOT expose endpoint connection details externally
- Do NOT disable connection monitoring

## Phase 3 — Rollback
- Restore previous security group rules
- Revert network configuration changes
- Restore endpoint configurations
- Verify connectivity after changes
- Document connection failure timeline and resolution
- Update connection troubleshooting procedures

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
