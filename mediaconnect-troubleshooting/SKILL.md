---
name: mediaconnect-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Elemental
  MediaConnect problems by analyzing flow creation, source connectivity,
  output delivery, VPC interfaces, encryption, transport streams,
  bandwidth, monitoring, maintenance events, multi-region flows,
  and following structured runbooks.
  Activate when: flow creation failures, source connectivity issues,
  output delivery problems, entitlement errors, VPC interface failures,
  encryption misconfigurations, transport stream errors, bandwidth
  constraints, monitoring gaps, maintenance events, multi-region
  flow issues, or the user says something is wrong with MediaConnect.
compatibility: >
  Requires AWS CLI or SDK access with mediaconnect, ec2, kms,
  cloudwatch, iam, and logs permissions.
---

# AWS Elemental MediaConnect Diagnostics

## When to use

Any AWS Elemental MediaConnect investigation — flow creation, source connectivity, output delivery, entitlements, VPC interfaces, encryption, transport stream errors, bandwidth issues, monitoring, maintenance events, or multi-region flow configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws mediaconnect list-flows --query 'Flows[*].{Name:Name,FlowArn:FlowArn,Status:Status,AvailabilityZone:AvailabilityZone}'
aws mediaconnect describe-flow --flow-arn <flow-arn>
aws cloudwatch get-metric-statistics --namespace AWS/MediaConnect --metric-name SourceBitRate --dimensions Name=FlowARN,Value=<flow-arn> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Average
```

### Step 2 — Domain deep dive

```
aws mediaconnect describe-flow --flow-arn <flow-arn> --query '{Source:Flow.Source,Outputs:Flow.Outputs,VpcInterfaces:Flow.VpcInterfaces}'
aws mediaconnect list-entitlements --query 'Entitlements[*].{Arn:EntitlementArn,DataTransferSubscriberFeePercent:DataTransferSubscriberFeePercent}'
aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].{GroupId:GroupId,IngressRules:IpPermissions}'
```

### Step 3 — Detailed investigation

```
aws cloudwatch get-metric-statistics --namespace AWS/MediaConnect --metric-name TransportStreamError --dimensions Name=FlowARN,Value=<flow-arn> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Sum
aws mediaconnect describe-flow --flow-arn <flow-arn> --query 'Flow.Source.Transport'
aws kms describe-key --key-id <key-id> --query '{KeyState:KeyMetadata.KeyState,KeyUsage:KeyMetadata.KeyUsage}'
```

Read `references/guardrails.md` before concluding on any MediaConnect issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `mediaconnect list-flows` | List all flows and their status |
| `mediaconnect describe-flow` | Get full flow details including source, outputs, VPC interfaces |
| `mediaconnect list-entitlements` | List entitlements for sharing flows |
| `mediaconnect start-flow` | Start a stopped flow |
| `mediaconnect stop-flow` | Stop a running flow |
| `cloudwatch get-metric-statistics` | Check MediaConnect metrics (bitrate, errors) |
| `ec2 describe-security-groups` | Verify security group rules for VPC interfaces |

## Gotchas: AWS Elemental MediaConnect

- MediaConnect flows are REGIONAL. A flow exists in a single AWS region and AZ. Multi-region requires separate flows with entitlements or gateway bridges.
- Source failover requires TWO sources in DIFFERENT availability zones. Both sources must deliver identical content with matching RTP sequence numbers.
- Entitlements share flow content with OTHER AWS accounts. The subscriber creates their own flow using the entitlement ARN. Entitlements do NOT share within the same account.
- VPC interfaces require subnets, security groups, and an Elastic Network Interface. Security groups must allow inbound traffic on the protocol port from the source IP range.
- MediaConnect supports SPEKE, static-key, and SRT encryption. SPEKE requires a DRM key provider URL. Static-key uses a Secrets Manager secret. SRT uses passphrase-based encryption.
- Transport stream (MPEG-TS) errors often indicate source-side issues — corrupted input, mismatched PID mappings, or discontinuity in the upstream encoder.
- Bandwidth is measured at the flow level. Output bandwidth is limited by the flow's max bitrate setting and the entitlement data transfer subscriber fee percentage.

## Anti-hallucination rules

1. Always cite specific flow ARNs, source names, output names, or API responses as evidence.
2. MediaConnect flows are regional and AZ-specific. Never claim a flow spans multiple regions automatically.
3. Source failover requires two sources in different AZs. Never suggest single-source failover.
4. Entitlements share between accounts, not within the same account. Never conflate with outputs.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Flow Management | A1-A2 | Flow creation failures, flow configuration |
| B — Source | B1-B2 | Source connectivity, source failover |
| C — Output | C1-C2 | Output failures, entitlement issues |
| D — Network & Security | D1-D2 | VPC interface errors, encryption issues |
| E — Transport & Performance | E1-E2 | Transport stream errors, bandwidth issues |
| F — Operations | F1-F2 | CloudWatch monitoring, maintenance events |
| G — Multi-Region | G1 | Multi-region flows |
| Z — Catch-All | Z1 | General troubleshooting |
