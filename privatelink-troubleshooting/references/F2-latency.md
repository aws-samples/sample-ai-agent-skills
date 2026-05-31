---
title: "F2 — Latency"
description: "Diagnose latency issues through VPC interface endpoints"
status: active
severity: MEDIUM
triggers:
  - "high latency"
  - "slow through endpoint"
  - "latency increase"
  - "endpoint latency"
  - "response time"
owner: devops-agent
objective: "Identify and resolve latency issues through interface endpoints"
context: "PrivateLink adds minimal latency (typically <1ms within the same AZ). Cross-AZ traffic adds ~1-2ms. Latency issues are usually caused by cross-AZ routing, DNS resolution delays, security group evaluation overhead, NLB target health (for endpoint services), or backend service performance. Compare latency with and without the endpoint."
---

## Phase 1 — Triage

MUST:
- Check if traffic is crossing AZs: compare client AZ with endpoint ENI AZ
- Verify endpoint state is available (degraded state can add latency)
- Check DNS resolution time from the client
- For endpoint services, check NLB target health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`

SHOULD:
- Measure latency with and without the endpoint to isolate the overhead
- Check NLB latency metrics: `aws cloudwatch get-metric-statistics --namespace AWS/NetworkELB --metric-name TargetResponseTime --dimensions Name=LoadBalancer,Value=<nlb-id> --start-time <start> --end-time <end> --period 300 --statistics Average,p99`
- Verify no packet loss through VPC Flow Logs
- Check client-side connection pooling and keep-alive settings

MAY:
- Use traceroute or mtr to identify latency hops
- Check for TCP retransmissions in network metrics
- Analyze application-level latency breakdown

## Phase 2 — Remediate

MUST:
- Place endpoint ENIs in the same AZ as latency-sensitive clients
- Ensure NLB targets are healthy (unhealthy targets cause failover latency)
- Fix DNS resolution issues (cache DNS, use local resolver)

SHOULD:
- Use AZ-specific DNS names for latency-sensitive workloads
- Enable connection pooling and keep-alive in the application
- Monitor latency metrics and set up alerts

MAY:
- Implement client-side DNS caching to reduce resolution overhead
- Consider connection pre-warming for latency-sensitive applications

## Common Issues

- symptoms: "~2ms additional latency through endpoint"
  diagnosis: "Cross-AZ traffic between client and endpoint ENI."
  resolution: "Place endpoint ENI in the same AZ as the client. Use AZ-specific DNS."

- symptoms: "Intermittent latency spikes"
  diagnosis: "DNS resolution delays or NLB target failover."
  resolution: "Cache DNS. Check NLB target health. Ensure healthy targets in all AZs."

- symptoms: "Latency higher than expected for same-AZ traffic"
  diagnosis: "Backend service performance issue, not endpoint overhead."
  resolution: "Check NLB target response time. Investigate backend service performance."

## Output Format

```yaml
root_cause: "latency — <specific_cause>"
evidence:
  - type: az_alignment
    content: "<client AZ vs endpoint ENI AZ>"
  - type: latency_metrics
    content: "<measured latency with and without endpoint>"
  - type: target_health
    content: "<NLB target health status>"
severity: MEDIUM
mitigation:
  immediate: "Optimize AZ placement and fix target health"
  long_term: "Implement latency monitoring and AZ-aware routing"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — latency is a performance metric
- Service Disruption Risk: MEDIUM — high latency affects application performance
- Compliance Impact: LOW — latency is an operational concern

## Escalation Conditions
- Latency exceeds SLA thresholds
- Latency spikes correlate with endpoint changes
- Cross-AZ latency significantly higher than expected
- Latency affects application timeout configurations
- Persistent latency degradation without clear cause

## Data Sensitivity
- Latency metrics — INTERNAL
- Performance baselines — INTERNAL
- Network path data — INTERNAL
- Application timeout configurations — INTERNAL
- Never expose latency data that reveals infrastructure details

## Prohibited Actions
- Do NOT ignore latency degradation on production endpoints
- Do NOT modify endpoint configurations without testing latency impact
- Do NOT bypass latency monitoring
- Do NOT change network routing without latency analysis
- Do NOT disable latency alerting
- Do NOT accept degraded latency without investigation

## Phase 3 — Rollback
- Restore previous endpoint configurations
- Revert network changes that increased latency
- Restore cross-AZ configurations
- Verify latency after changes
- Document latency issues and performance baselines
- Update latency monitoring thresholds

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
