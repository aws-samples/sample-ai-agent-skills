---
title: "D2 — Wavelength Throughput Issues"
description: "Diagnose throughput limitations in Wavelength Zone deployments"
status: active
severity: MEDIUM
triggers:
  - "wavelength throughput"
  - "bandwidth wavelength"
  - "slow transfer"
  - "network performance wavelength"
owner: devops-agent
objective: "Identify and resolve throughput bottlenecks in Wavelength Zone deployments"
context: "Throughput in Wavelength Zones depends on instance type network performance, carrier network capacity, and application architecture. The carrier network path may have different bandwidth characteristics than Region networking."
---

## Phase 1 — Triage

MUST:
- Check instance type network performance: `aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[*].NetworkInfo'`
- Monitor network metrics: `aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name NetworkIn --dimensions Name=InstanceId,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check NetworkOut: `aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name NetworkOut --dimensions Name=InstanceId,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify enhanced networking is enabled on the instance

SHOULD:
- Check for network throttling indicators
- Verify security group rules are not limiting traffic
- Test throughput between Wavelength and Region instances

MAY:
- Use iperf3 for bandwidth testing between endpoints
- Check carrier network utilization in the area

## Phase 2 — Remediate

MUST:
- If instance type limiting: upgrade to a type with higher network bandwidth
- If enhanced networking not enabled: enable it on the instance
- If security groups limiting: adjust rules for required traffic

SHOULD:
- Use instances with dedicated network bandwidth for throughput-sensitive workloads
- Implement connection pooling and multiplexing
- Optimize data transfer patterns (compression, batching)

MAY:
- Implement application-level traffic shaping
- Consider multiple instances for horizontal scaling of throughput

## Common Issues

- symptoms: "Throughput capped at a specific bandwidth"
  diagnosis: "Instance type network bandwidth limit reached."
  resolution: "Upgrade to an instance type with higher network performance."

- symptoms: "Throughput to Region services is lower than expected"
  diagnosis: "Carrier-to-Region path has limited bandwidth or high utilization."
  resolution: "Minimize Region dependencies. Cache data locally on Wavelength instances."

## Output Format

```yaml
root_cause: "throughput — <specific_cause>"
evidence:
  - type: network_metrics
    content: "<NetworkIn/NetworkOut data>"
  - type: instance_network
    content: "<instance type network capabilities>"
severity: MEDIUM
mitigation:
  immediate: "Upgrade instance type or fix network configuration"
  long_term: "Optimize architecture for Wavelength throughput patterns"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Edge deployment configurations reveal application architecture
- Carrier IP assignments expose public-facing endpoints
- Latency data may reveal user geographic distribution
- Classification: INTERNAL - edge infrastructure topology and carrier network details

## Prohibited Actions

- NEVER terminate EC2 instances in Wavelength zones serving live edge traffic
- NEVER modify carrier gateway configurations during peak usage
- NEVER change VPC subnet associations without confirming application connectivity
- NEVER remove carrier IP addresses assigned to production workloads

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
