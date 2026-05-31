---
name: globalaccelerator-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Global Accelerator problems
  by analyzing accelerator configurations, endpoint health, routing policies,
  performance metrics, DNS resolution, and following structured runbooks.
  Activate when: accelerator creation failures, static IP issues, endpoint
  health check failures, endpoint group configuration, traffic dial problems,
  client affinity issues, flow log analysis, latency problems, throughput
  degradation, DNS resolution failures, custom routing issues, DDoS protection
  concerns, cross-account endpoint problems, ALB/NLB/EC2/EIP integration
  issues, or the user says something is wrong with Global Accelerator
  without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Global Accelerator, EC2, ELB, CloudWatch,
  CloudTrail, IAM, and optionally Shield and VPC Flow Logs permissions.
---

# Global Accelerator Diagnostics

## When to use

Any Global Accelerator investigation where the console alone is insufficient — accelerator configuration analysis, endpoint health debugging, routing policy troubleshooting, performance optimization, DNS resolution issues, custom routing configuration, or security concerns.

## Investigation workflow

### Step 1 — Collect and triage

```
aws globalaccelerator list-accelerators
aws globalaccelerator describe-accelerator --accelerator-arn <accelerator-arn>
aws globalaccelerator describe-accelerator-attributes --accelerator-arn <accelerator-arn>
aws cloudwatch get-metric-statistics --namespace AWS/GlobalAccelerator --metric-name NewFlowCount --dimensions Name=Accelerator,Value=<accelerator-arn> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/GlobalAccelerator --metric-name ProcessedBytesIn --dimensions Name=Accelerator,Value=<accelerator-arn> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws globalaccelerator list-listeners --accelerator-arn <accelerator-arn>
aws globalaccelerator list-endpoint-groups --listener-arn <listener-arn>
aws globalaccelerator describe-endpoint-group --endpoint-group-arn <endpoint-group-arn>
aws globalaccelerator list-custom-routing-accelerators
aws globalaccelerator describe-custom-routing-accelerator --accelerator-arn <accelerator-arn>
```

### Step 3 — Detailed investigation

```
aws globalaccelerator describe-listener --listener-arn <listener-arn>
aws globalaccelerator list-custom-routing-port-mappings --accelerator-arn <accelerator-arn>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=globalaccelerator.amazonaws.com
aws ec2 describe-flow-logs --filter Name=resource-id,Values=<vpc-id>
```

Read `references/globalaccelerator-guardrails.md` before concluding on any Global Accelerator issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-accelerator` | Full accelerator details and status |
| `describe-accelerator-attributes` | Flow logs and other attributes |
| `list-listeners` | Listener port/protocol configuration |
| `list-endpoint-groups` | Endpoint groups per listener |
| `describe-endpoint-group` | Health checks, traffic dial, endpoints |
| `list-custom-routing-accelerators` | Custom routing accelerators |
| `list-custom-routing-port-mappings` | Port-to-instance mappings |
| `describe-custom-routing-endpoint-group` | Custom routing endpoint group |
| CloudWatch metrics | Flow counts, processed bytes, healthy endpoints |
| CloudTrail | Configuration change audit |

## Gotchas: Global Accelerator

- Static anycast IPs: each accelerator gets exactly 2 static anycast IP addresses. These IPs are allocated from the AWS global network and cannot be changed after creation. They persist for the lifetime of the accelerator.
- Endpoint health checks are independent of ELB health checks. Global Accelerator performs its own health checks on endpoints. An endpoint can be healthy in ELB but unhealthy in Global Accelerator if the GA health check port/path differs.
- Traffic dial controls the percentage of traffic routed to an endpoint group (regional). It does NOT control traffic to individual endpoints within a group. Use endpoint weights for per-endpoint traffic distribution.
- Client affinity is per-accelerator listener (not per-endpoint). When enabled, all requests from a client IP go to the same endpoint. Options are NONE (default), SOURCE_IP. Client affinity applies to the listener, not individual endpoint groups.
- Custom routing maps specific ports on the accelerator to specific EC2 instances in a VPC subnet. It does NOT support ALB/NLB endpoints. Custom routing accelerators and standard accelerators are different resource types.
- Global Accelerator is NOT a CDN — it does not cache content. It optimizes the network path using the AWS global network. For caching, use CloudFront. GA and CloudFront serve different purposes and can be used together.
- DDoS protection via AWS Shield Standard is included automatically. Shield Advanced can be associated for enhanced protection and DDoS response team access.
- Endpoint types: ALB, NLB, EC2 instances, and Elastic IP addresses. Each endpoint group can contain up to 10 endpoints. Endpoints must be in the same region as the endpoint group.
- Cross-account endpoints: Global Accelerator supports endpoints in different AWS accounts using cross-account attachment resources.
- Flow logs: Global Accelerator flow logs capture information about IP address traffic going to and from network interfaces in your accelerator. Logs are published to S3.

## Anti-hallucination rules

1. Always cite specific accelerator configurations, endpoint health status, or API responses as evidence.
2. Global Accelerator is NOT a CDN. Never claim it caches content or replaces CloudFront.
3. Traffic dial is per-endpoint-group (regional), NOT per-endpoint. Never confuse with endpoint weights.
4. Client affinity is per-listener, NOT per-endpoint or per-endpoint-group.
5. Custom routing accelerators are separate from standard accelerators. Never mix their APIs.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 22 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Accelerator | A1-A3 | Creation failures, static IP issues, configuration |
| B — Endpoints | B1-B3 | Endpoint health, endpoint groups, traffic dial |
| C — Routing | C1-C2 | Client affinity, flow logs |
| D — Performance | D1-D2 | Latency, throughput |
| E — DNS | E1-E2 | DNS resolution, custom routing |
| F — Security | F1-F2 | DDoS protection, cross-account endpoints |
| G — Integration | G1-G2 | ALB/NLB/EC2/EIP endpoints, custom routing accelerator |
| Z — Catch-All | Z1 | General troubleshooting |
