---
title: "B1 — Source Connectivity"
description: "Diagnose MediaConnect source connectivity and ingest failures"
status: active
severity: CRITICAL
triggers: ["no source signal", "source disconnected", "source bitrate zero", "ingest failure", "source not receiving"]
owner: devops-agent
objective: "Restore source connectivity and verify video ingest is functioning"
context: "Source connectivity depends on protocol, network path, firewall rules, and sender configuration. VPC sources require ENI and security group access."
---

## Phase 1 — Triage

MUST:
- Check flow status and source details: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query '{Status:Flow.Status,Source:Flow.Source}'`
- Check source bitrate metric: `aws cloudwatch get-metric-statistics --namespace AWS/MediaConnect --metric-name SourceBitRate --dimensions Name=FlowARN,Value=<flow-arn> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Average`
- Verify flow is ACTIVE: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query 'Flow.Status'`

SHOULD:
- Check source whitelist CIDR matches sender IP
- Verify protocol and port match between sender and MediaConnect source
- For VPC sources, check security group inbound rules: `aws ec2 describe-security-groups --group-ids <sg-id>`

MAY:
- Check network ACLs on the VPC subnet
- Verify the sender is actively transmitting
- Review CloudWatch SourcePacketLossPercent metric

## Phase 2 — Remediate

MUST:
- Ensure the flow is in ACTIVE state: `aws mediaconnect start-flow --flow-arn <flow-arn>`
- Fix source whitelist CIDR to include sender IP: `aws mediaconnect update-flow-source --flow-arn <flow-arn> --source-arn <source-arn> --whitelist-cidr <sender-cidr>`
- For VPC sources, update security group to allow inbound on protocol port: `aws ec2 authorize-security-group-ingress --group-id <sg-id> --protocol udp --port <port> --cidr <sender-cidr>`

SHOULD:
- Verify sender is transmitting to the correct MediaConnect ingest IP and port
- Confirm protocol settings match (SRT caller/listener, Zixi push/pull)

MAY:
- Use traceroute or MTR from sender to MediaConnect ingest endpoint to diagnose network path issues
- Enable VPC Flow Logs for packet-level analysis

## Safety Ratings

```
safety_ratings:
  - "describe-flow: GREEN — read-only flow inspection"
  - "get-metric-statistics: GREEN — read-only CloudWatch query"
  - "update-flow-source: YELLOW — modifies source configuration on a potentially live flow"
  - "authorize-security-group-ingress: YELLOW — opens network access, verify CIDR is specific"
```

## Escalation Conditions

- Source bitrate remains zero after verifying all network and protocol settings
- VPC flow logs show packets arriving but MediaConnect reports no source
- Source connectivity is intermittent with no pattern in CloudWatch metrics
- Multiple flows in the same region simultaneously lose source connectivity

## Data Sensitivity

- `aws mediaconnect describe-flow`: MEDIUM — contains source IPs, encryption settings, VPC details
- `aws cloudwatch get-metric-statistics`: LOW — returns numeric metric values only
- `aws ec2 describe-security-groups`: MEDIUM — reveals network access rules
- `aws ec2 authorize-security-group-ingress`: HIGH — modifies network access controls

## Prohibited Actions

- NEVER open security group to 0.0.0.0/0 for troubleshooting — use specific sender CIDR
- NEVER modify source whitelist CIDR to 0.0.0.0/0 in production
- NEVER delete and recreate a flow to fix source connectivity — update the source instead
- NEVER disable encryption to test connectivity on a production flow

## Phase 3 — Rollback

1. If security group was opened too broadly, revoke the rule: `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol udp --port <port> --cidr <overly-broad-cidr>`
2. If source whitelist was changed incorrectly, revert: `aws mediaconnect update-flow-source --flow-arn <flow-arn> --source-arn <source-arn> --whitelist-cidr <original-cidr>`
3. If flow was stopped during troubleshooting, restart: `aws mediaconnect start-flow --flow-arn <flow-arn>`
4. Verify source bitrate has recovered via CloudWatch metrics
5. Document the connectivity issue and resolution for the network operations team

## Common Issues

- symptoms: "Source bitrate is zero but flow is ACTIVE"
  diagnosis: "Sender is not transmitting, or network path is blocked."
  resolution: "Verify sender is active and transmitting to the correct IP:port. Check firewalls and security groups."

- symptoms: "Source bitrate is intermittent — drops to zero periodically"
  diagnosis: "Network instability between sender and MediaConnect, or sender encoder issues."
  resolution: "Check network path quality with MTR. Verify sender encoder stability. Consider SRT protocol for lossy networks."

- symptoms: "VPC source shows no connectivity despite correct security group"
  diagnosis: "Network ACL on the subnet may be blocking traffic, or ENI is in wrong subnet."
  resolution: "Check network ACLs allow inbound/outbound on the protocol port. Verify ENI subnet matches expected network path."

## Output Format

```yaml
root_cause: "source_connectivity — <specific_cause>"
evidence:
  - type: metric_data
    content: "<SourceBitRate metric showing zero or drops>"
severity: CRITICAL
mitigation:
  immediate: "Restore source connectivity"
  long_term: "Implement source failover with dual-AZ sources"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
