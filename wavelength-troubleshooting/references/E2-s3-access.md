---
title: "E2 — S3 Access from Wavelength"
description: "Diagnose S3 access issues from Wavelength Zone instances"
status: active
severity: MEDIUM
triggers:
  - "S3 wavelength"
  - "S3 access"
  - "object storage wavelength"
  - "S3 slow wavelength"
owner: devops-agent
objective: "Resolve S3 access and performance issues from Wavelength Zone instances"
context: "S3 is not available directly in Wavelength Zones. All S3 access routes through the parent Region VPC. Using S3 gateway endpoints optimizes this path. S3 access from Wavelength inherently has higher latency than Region-local access."
---

## Phase 1 — Triage

MUST:
- Check for S3 gateway endpoint: `aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=<vpc-id>" "Name=service-name,Values=com.amazonaws.<region>.s3"`
- Verify route table includes S3 endpoint route: `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<wavelength-subnet-id>"`
- Check IAM permissions for S3 access from the instance role
- Test S3 access: verify the instance can list buckets or get objects

SHOULD:
- Check S3 endpoint policy if using VPC endpoint
- Verify bucket policy allows access from the VPC or endpoint
- Measure S3 access latency from the Wavelength instance

MAY:
- Check for S3 Transfer Acceleration as an alternative
- Review application S3 access patterns for optimization

## Phase 2 — Remediate

MUST:
- If no S3 endpoint: create gateway endpoint: `aws ec2 create-vpc-endpoint --vpc-id <vpc-id> --service-name com.amazonaws.<region>.s3 --route-table-ids <rtb-id>`
- If permission denied: fix IAM role or bucket policy
- If endpoint policy restricting: update endpoint policy

SHOULD:
- Use S3 gateway endpoint for all S3 traffic (no cost, better performance)
- Implement local caching for frequently accessed S3 objects
- Use multipart uploads for large objects

MAY:
- Implement application-level caching (Redis, local disk) for S3 data
- Use S3 Transfer Acceleration for cross-Region access patterns

## Common Issues

- symptoms: "S3 access from Wavelength instance times out"
  diagnosis: "No route to S3 — missing VPC endpoint or NAT gateway."
  resolution: "Create S3 gateway endpoint and associate with Wavelength subnet route table."

- symptoms: "S3 access works but is slow from Wavelength"
  diagnosis: "S3 traffic routes through Region — inherent latency for Wavelength-to-Region path."
  resolution: "Use S3 gateway endpoint. Implement local caching for frequently accessed data."

## Output Format

```yaml
root_cause: "s3_access — <specific_cause>"
evidence:
  - type: vpc_endpoint
    content: "<S3 endpoint configuration>"
  - type: route_table
    content: "<route table entries>"
severity: MEDIUM
mitigation:
  immediate: "Create S3 gateway endpoint or fix permissions"
  long_term: "Implement local caching to minimize S3 round-trips"
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
