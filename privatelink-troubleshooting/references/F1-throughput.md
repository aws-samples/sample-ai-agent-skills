---
title: "F1 — Throughput"
description: "Diagnose throughput limitations through VPC interface endpoints"
status: active
severity: MEDIUM
triggers:
  - "throughput limit"
  - "bandwidth through endpoint"
  - "slow transfer"
  - "endpoint throughput"
  - "data transfer speed"
owner: devops-agent
objective: "Identify and resolve throughput limitations through interface endpoints"
context: "Interface endpoints use ENIs which have bandwidth limits based on the instance type of the client. Each endpoint ENI supports up to 10 Gbps per AZ (burst to 40 Gbps with enhanced networking). Throughput is shared across all connections through the ENI. Cross-AZ traffic adds latency. NLB throughput limits apply for endpoint services."
---

## Phase 1 — Triage

MUST:
- Check endpoint ENI placement and count: `aws ec2 describe-network-interfaces --filters Name=vpc-endpoint-id,Values=<endpoint-id> --query 'NetworkInterfaces[*].{Id:NetworkInterfaceId,AZ:AvailabilityZone}'`
- Check client instance type and network performance
- Monitor network throughput metrics on client instances: `aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name NetworkOut --dimensions Name=InstanceId,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check for cross-AZ traffic patterns

SHOULD:
- Check NLB metrics for endpoint services: `aws cloudwatch get-metric-statistics --namespace AWS/NetworkELB --metric-name ProcessedBytes --dimensions Name=LoadBalancer,Value=<nlb-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify client is using the closest AZ endpoint ENI
- Check for connection limits on the endpoint

MAY:
- Run throughput tests using iperf3 or similar tools
- Analyze VPC Flow Logs for traffic volume patterns

## Phase 2 — Remediate

MUST:
- Ensure endpoint ENIs are in the same AZ as high-throughput clients
- Use instance types with sufficient network bandwidth for the client
- Distribute traffic across multiple AZs if single AZ is saturated

SHOULD:
- Enable cross-zone load balancing on NLB for endpoint services
- Use AZ-specific DNS names to route traffic to local endpoint ENIs
- Monitor throughput metrics and set up alerts

MAY:
- Consider multiple connections for parallel data transfer
- Evaluate if the workload needs a different architecture (e.g., S3 gateway endpoint for bulk data)

## Common Issues

- symptoms: "Throughput limited to ~5 Gbps through endpoint"
  diagnosis: "Client instance type has limited network bandwidth."
  resolution: "Use an instance type with higher network performance (e.g., c5n, m5n families)."

- symptoms: "Throughput drops during peak hours"
  diagnosis: "Shared bandwidth on the endpoint ENI or client instance."
  resolution: "Distribute traffic across multiple AZs. Use larger instance types."

- symptoms: "NLB throughput limit for endpoint service"
  diagnosis: "NLB target group or targets cannot handle the throughput."
  resolution: "Scale NLB targets. Enable cross-zone load balancing. Add targets in more AZs."

## Output Format

```yaml
root_cause: "throughput — <specific_cause>"
evidence:
  - type: endpoint_enis
    content: "<ENI count and AZ distribution>"
  - type: network_metrics
    content: "<throughput metrics from CloudWatch>"
  - type: client_instance
    content: "<client instance type and network capability>"
severity: MEDIUM
mitigation:
  immediate: "Optimize AZ placement and instance types"
  long_term: "Implement throughput monitoring and capacity planning"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — throughput is a performance concern
- Service Disruption Risk: MEDIUM — throughput issues affect application performance
- Compliance Impact: LOW — throughput is an operational concern

## Escalation Conditions
- Throughput drops below acceptable thresholds
- Throughput bottleneck identified at endpoint level
- Cross-AZ traffic causing throughput issues
- NLB target health affecting throughput
- Throughput degradation during peak hours

## Data Sensitivity
- Throughput metrics — INTERNAL
- Traffic patterns — INTERNAL
- NLB configurations — INTERNAL
- Performance baselines — INTERNAL
- Never expose throughput data that reveals business volume

## Prohibited Actions
- Do NOT ignore throughput degradation on production endpoints
- Do NOT modify NLB configurations without testing
- Do NOT bypass throughput monitoring
- Do NOT change endpoint configurations during peak hours without approval
- Do NOT ignore cross-AZ throughput patterns
- Do NOT disable throughput monitoring

## Phase 3 — Rollback
- Restore previous endpoint configurations
- Revert NLB changes that affected throughput
- Restore cross-AZ configurations
- Verify throughput after changes
- Document throughput issues and performance baselines
- Update capacity planning based on findings

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
