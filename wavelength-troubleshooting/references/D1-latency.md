---
title: "D1 — Wavelength Latency Issues"
description: "Diagnose high latency problems in Wavelength Zone deployments"
status: active
severity: HIGH
triggers:
  - "wavelength latency"
  - "high latency"
  - "slow response"
  - "latency optimization"
owner: devops-agent
objective: "Identify and resolve sources of latency in Wavelength Zone deployments"
context: "Wavelength is designed for single-digit millisecond latency from mobile devices. High latency may be caused by routing through the Region, carrier network conditions, application architecture, or DNS resolution overhead."
---

## Phase 1 — Triage

MUST:
- Verify instance is in Wavelength Zone: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].Placement.AvailabilityZone'`
- Check carrier IP is associated (direct mobile path): `aws ec2 describe-addresses --filters "Name=instance-id,Values=<instance-id>" --query 'Addresses[?CarrierIp!=null]'`
- Verify carrier gateway route: `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<subnet-id>"`
- Check instance network metrics: `aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name NetworkIn --dimensions Name=InstanceId,Value=<instance-id> --start-time <start> --end-time <end> --period 60 --statistics Average`

SHOULD:
- Measure end-to-end latency from mobile device to Wavelength instance
- Check if traffic is routing through the Region instead of directly to Wavelength
- Verify DNS resolution is not adding latency (Region DNS resolver)
- Check application-level processing time

MAY:
- Use traceroute from mobile device to identify latency sources
- Check carrier network conditions and congestion
- Review application architecture for unnecessary Region hops

## Phase 2 — Remediate

MUST:
- If traffic routing through Region: fix route table to use carrier gateway
- If no carrier IP: allocate and associate for direct mobile path
- If DNS latency: implement local DNS caching on the instance

SHOULD:
- Minimize dependencies on Region services for latency-sensitive paths
- Cache frequently accessed data locally on the Wavelength instance
- Use connection pooling to reduce connection setup overhead

MAY:
- Implement edge caching for static content
- Optimize application code for low-latency operation
- Consider instance type with better network performance

## Common Issues

- symptoms: "Latency from mobile device is 50ms+ instead of single-digit"
  diagnosis: "Traffic routing through Region instead of directly to Wavelength Zone."
  resolution: "Verify carrier gateway route and carrier IP association."

- symptoms: "First request is slow, subsequent requests are fast"
  diagnosis: "DNS resolution going to Region resolver or TCP connection setup overhead."
  resolution: "Implement local DNS caching. Use connection keep-alive."

## Output Format

```yaml
root_cause: "latency — <specific_cause>"
evidence:
  - type: routing_path
    content: "<traffic routing analysis>"
  - type: latency_measurements
    content: "<end-to-end latency data>"
severity: HIGH
mitigation:
  immediate: "Fix routing or DNS to reduce latency"
  long_term: "Optimize architecture for Wavelength low-latency patterns"
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
