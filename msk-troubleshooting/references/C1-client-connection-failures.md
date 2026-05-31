---
title: "C1 — MSK Client Connection Failures"
description: "Diagnose and resolve client connection failures to MSK brokers"
status: active
severity: HIGH
triggers:
  - "connection refused"
  - "connection timeout"
  - "cannot connect"
  - "broker unreachable"
  - "bootstrap server"
  - "connection failed"
owner: devops-agent
objective: "Identify why clients cannot connect to MSK brokers and restore connectivity"
context: "MSK brokers run in private subnets within a VPC. Clients must have network connectivity to the broker endpoints. Connection failures are commonly caused by wrong bootstrap server endpoints, incorrect ports for the auth method, security group rules blocking traffic, clients outside the VPC without peering/VPN, DNS resolution failures, or TLS configuration mismatches."
---

## Phase 1 — Triage

MUST:
- Get bootstrap broker endpoints: `aws kafka get-bootstrap-brokers --cluster-arn <cluster-arn>`
- Check cluster status: `aws kafka describe-cluster-v2 --cluster-arn <cluster-arn> --query 'ClusterInfo.{State:State,StateInfo:StateInfo}'`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <broker-sg-id> --query 'SecurityGroups[*].IpPermissions'`
- Verify client is in the same VPC or has connectivity (peering, VPN, Transit Gateway)

SHOULD:
- Check client security group allows outbound to broker ports
- Verify DNS resolution of broker endpoints from client: test with `nslookup <broker-endpoint>` or `dig <broker-endpoint>`
- Check if client is using the correct port for the auth method (9092/9094/9096/9098)
- Check subnet route tables for connectivity between client and broker subnets

MAY:
- Test TCP connectivity: `nc -zv <broker-endpoint> <port>` or `telnet <broker-endpoint> <port>`
- Check VPC flow logs for rejected traffic
- Verify NACLs are not blocking traffic on broker ports

## Phase 2 — Remediate

MUST:
- If wrong port: use correct port for auth method (9098=IAM, 9096=SASL/SCRAM, 9094=mTLS, 9092=plaintext)
- If security group blocks: add inbound rule allowing TCP on required ports from client CIDR/SG
- If client outside VPC: set up VPC peering, VPN, Transit Gateway, or PrivateLink
- If DNS not resolving: ensure VPC DNS resolution is enabled and client uses VPC DNS

SHOULD:
- Open all broker ports (9092-9098) and inter-broker port (9093) in security groups
- Use the bootstrap broker string from `get-bootstrap-brokers` — do not construct manually
- Ensure client TLS configuration matches broker TLS settings

MAY:
- Enable multi-VPC connectivity for cross-VPC access
- Set up VPC endpoints for MSK
- Configure client connection timeout and retry settings

## Common Issues

- symptoms: "Connection timeout from client in different VPC"
  diagnosis: "No VPC peering, Transit Gateway, or VPN between client VPC and MSK VPC."
  resolution: "Set up VPC peering or Transit Gateway. Update route tables and security groups."

- symptoms: "Connection refused on port 9092 but cluster uses IAM auth"
  diagnosis: "Port 9092 is for plaintext/unauthenticated. IAM auth uses port 9098."
  resolution: "Use the IAM bootstrap broker string (port 9098) from get-bootstrap-brokers."

- symptoms: "Intermittent connection failures after initial success"
  diagnosis: "Client connects to bootstrap brokers but cannot reach individual brokers for partition leaders."
  resolution: "Ensure security groups allow traffic on all broker ports, not just the bootstrap port."

## Output Format

```yaml
root_cause: "connection_failure — <specific_cause>"
evidence:
  - type: bootstrap_brokers
    content: "<bootstrap broker endpoints>"
  - type: security_groups
    content: "<inbound rules on broker security group>"
  - type: network_test
    content: "<connectivity test results>"
severity: HIGH
mitigation:
  immediate: "Fix security group rules or network connectivity"
  long_term: "Document correct endpoints and ports, automate security group management"
```


## Safety Ratings
```
safety_ratings:
  - "Get bootstrap broker endpoints: GREEN — read-only API call"
  - "Check security group rules: GREEN — read-only network inspection"
  - "Verify DNS resolution: GREEN — read-only network test"
  - "Add security group inbound rules: YELLOW — changes network access, may expose brokers to unintended traffic"
  - "Set up VPC peering: YELLOW — creates network path between VPCs, security implications"
  - "Modify route tables: YELLOW — changes network routing, can affect other services in the VPC"
```

## Escalation Conditions
- Cluster serves production event streaming
- Connection failures affecting production applications
- Fix requires VPC peering or Transit Gateway changes (cross-team coordination)
- Security group changes may impact other services sharing the same groups
- DNS resolution failures suggesting VPC-level networking issues

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Bootstrap broker endpoints: cluster access points"
    - "Security group rules: network access control configuration"
    - "VPC CIDR ranges: network topology"
    - "Subnet IDs and route tables: infrastructure layout"
  handling: "Do not expose VPC topology, CIDR ranges, or security group details externally."
```

## Prohibited Actions
- NEVER suggest deleting topics with active consumers
- NEVER suggest reducing broker count
- NEVER open broker ports to 0.0.0.0/0 (all traffic)
- NEVER remove existing security group rules without understanding what other services depend on them
- NEVER manually construct bootstrap broker strings — always use get-bootstrap-brokers API

## Phase 3 — Rollback
- If security group rules were added: remove the added inbound/outbound rules to restore previous access controls
- If VPC peering was created: delete the peering connection and remove associated route table entries
- If route table entries were added: remove the added routes to restore previous routing
- If NACLs were modified: revert NACL rules to previous configuration (both inbound and outbound)
- If DNS settings were changed: restore previous VPC DNS configuration

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing authentication"
  - "NEVER suggest public access to brokers"
