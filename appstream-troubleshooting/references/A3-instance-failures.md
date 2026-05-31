---
title: "A3 — Fleet Instance Failures"
description: "Diagnose AppStream 2.0 fleet instance failures"
status: active
severity: HIGH
triggers:
  - "instance failure"
  - "fleet instance unhealthy"
  - "instance not starting"
  - "fleet capacity not reaching desired"
owner: devops-agent
objective: "Identify and resolve fleet instance failures"
context: "Fleet instances can fail due to image issues, VPC networking problems, security group restrictions, instance type capacity, or service health issues. Failed instances reduce available capacity and may cause InsufficientCapacityError."
---

## Phase 1 — Triage

MUST:
- Check fleet status and capacity: `aws appstream describe-fleets --names <fleet-name>`
- Verify running vs desired capacity (running < desired indicates failures)
- Check CloudWatch for fleet errors: `aws cloudwatch get-metric-statistics --namespace AWS/AppStream --metric-name RunningCapacity --dimensions Name=Fleet,Value=<fleet-name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check CloudTrail for fleet-related errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=appstream.amazonaws.com --max-results 20`

SHOULD:
- Verify the image is still available and not corrupted
- Check VPC subnet capacity (available IP addresses)
- Verify security groups allow required outbound traffic
- Check if the instance type has capacity in the AZ

MAY:
- Check AWS Health Dashboard for AppStream service issues
- Review fleet event logs in the AppStream console

## Phase 2 — Remediate

MUST:
- Fix VPC networking issues (subnets, security groups, NAT gateway)
- Ensure the image is valid and available
- Verify instance type availability in the region

SHOULD:
- Use multiple subnets across AZs for resilience
- Ensure subnets have sufficient IP addresses
- Allow outbound HTTPS (443) in security groups for AppStream service communication

MAY:
- Try a different instance type if capacity is limited
- Contact AWS Support for persistent instance failures

## Common Issues

- symptoms: "Running capacity consistently below desired"
  diagnosis: "Instances failing to start due to networking or image issues."
  resolution: "Check VPC subnets, security groups, and image status."

- symptoms: "Instances start but become unhealthy quickly"
  diagnosis: "Application on the image causing crashes or health check failures."
  resolution: "Test the image with an image builder. Check application compatibility."

- symptoms: "Capacity issues in specific AZ"
  diagnosis: "Instance type capacity exhausted in one AZ."
  resolution: "Add subnets in additional AZs. Consider a different instance type."

## Output Format

```yaml
root_cause: "instance_failure — <specific_cause>"
evidence:
  - type: fleet_capacity
    content: "<running vs desired capacity>"
  - type: vpc_config
    content: "<VPC subnet and security group status>"
  - type: image_status
    content: "<image availability>"
severity: HIGH
mitigation:
  immediate: "Fix the instance failure cause"
  long_term: "Multi-AZ deployment with monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix VPC networking issues | YELLOW | Network configuration change |
| Ensure image is valid and available | GREEN | Verification |
| Verify instance type availability | GREEN | Verification |
| Use multiple subnets across AZs | GREEN | Best practice |
| Ensure sufficient IP addresses | GREEN | Capacity check |
| Allow outbound HTTPS in security groups | YELLOW | Security group change |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **Fleet capacity**: Running vs desired instance counts — INTERNAL
- **VPC configuration**: Subnet and security group details — INTERNAL
- **Image status**: Image availability and health — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER modify security groups without understanding impact on active sessions
- NEVER change instance types on a fleet with active sessions

## Phase 3 — Rollback

1. Revert VPC configuration changes (subnets, security groups)
2. Revert to previous image if image change caused failures
3. Restore previous instance type if it was changed
4. Verify fleet capacity recovers to desired level
5. Monitor instance health for 30 minutes after rollback

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
