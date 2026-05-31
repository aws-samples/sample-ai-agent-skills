---
title: "A2 — Registration Issues"
description: "Diagnose and resolve ELB target registration and deregistration issues"
status: active
severity: HIGH
triggers:
  - "target registration failed"
  - "target not registered"
  - "deregistration timeout"
  - "target draining"
owner: devops-agent
objective: "Resolve target registration failures and ensure proper target lifecycle management"
context: "Targets must be registered with a target group to receive traffic from the load balancer. Registration can fail due to incorrect target type (instance, IP, Lambda), cross-AZ restrictions, VPC mismatches, or API permission issues. Deregistration involves a configurable draining period during which in-flight requests complete. Issues arise during auto-scaling events, deployments, and manual target management."
---

## Phase 1 — Triage

MUST:
- List registered targets: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Check target group type: `aws elbv2 describe-target-groups --target-group-arns <tg-arn> --query 'TargetGroups[0].TargetType'`
- Verify the target exists and is in the correct VPC: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[0].Instances[0].{VPC:VpcId,State:State.Name,AZ:Placement.AvailabilityZone}'`
- Check CloudTrail for registration errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RegisterTargets --max-results 10`
- Verify IAM permissions for registration: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names elasticloadbalancing:RegisterTargets`

SHOULD:
- Check if the target group is associated with a load balancer: `aws elbv2 describe-target-groups --target-group-arns <tg-arn> --query 'TargetGroups[0].LoadBalancerArns'`
- Verify deregistration delay setting: `aws elbv2 describe-target-group-attributes --target-group-arn <tg-arn> --query 'Attributes[?Key==\`deregistration_delay.timeout_seconds\`]'`
- Check Auto Scaling group target group associations if applicable

MAY:
- Review the target group's tags for environment identification
- Check if Lambda function targets have the correct invoke permissions
- Verify IP-type targets are in the correct VPC or peered VPC

## Phase 2 — Remediate

MUST:
- If target type mismatch: register with the correct target type (instance ID for instance type, IP for IP type)
- If VPC mismatch: register targets in the same VPC as the target group (or use IP type for cross-VPC)
- If permission denied: add `elasticloadbalancing:RegisterTargets` to the IAM policy
- If target stuck in draining: wait for deregistration delay to expire, or reduce the delay: `aws elbv2 modify-target-group-attributes --target-group-arn <tg-arn> --attributes Key=deregistration_delay.timeout_seconds,Value=<seconds>`

SHOULD:
- Register targets: `aws elbv2 register-targets --target-group-arn <tg-arn> --targets Id=<target-id>,Port=<port>`
- Verify registration succeeded: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- For Auto Scaling: verify the ASG is configured with the correct target group ARN

MAY:
- Implement automated target registration via Auto Scaling lifecycle hooks
- Set up CloudWatch alarms for HealthyHostCount dropping below threshold
- Document the target registration process for the team

## Common Issues

- symptoms: "RegisterTargets returns InvalidTarget error"
  diagnosis: "Target ID format doesn't match the target group type (e.g., instance ID for IP-type group)."
  resolution: "Use the correct target format: instance ID for instance type, IP address for IP type."

- symptoms: "Target stuck in 'draining' state for too long"
  diagnosis: "Deregistration delay is set too high or long-lived connections are keeping the target active."
  resolution: "Reduce deregistration delay or terminate long-lived connections."

- symptoms: "Auto Scaling launches instances but they don't appear in target group"
  diagnosis: "ASG not associated with the target group."
  resolution: "Attach the target group to the ASG: `aws autoscaling attach-load-balancer-target-groups --auto-scaling-group-name <asg> --target-group-arns <tg-arn>`."

- symptoms: "Lambda target returns 'permission denied'"
  diagnosis: "ELB does not have invoke permission on the Lambda function."
  resolution: "Add Lambda permission: `aws lambda add-permission --function-name <fn> --statement-id elb --action lambda:InvokeFunction --principal elasticloadbalancing.amazonaws.com`."

## Output Format

```yaml
root_cause: "Registration issue — <specific_cause>"
evidence:
  - type: target_health
    content: "<describe-target-health output>"
  - type: target_group_config
    content: "<target group type and configuration>"
severity: HIGH
mitigation:
  immediate: "Fix target type, permissions, or registration configuration"
  long_term: "Automate target registration via Auto Scaling and IaC"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/list): GREEN — read-only"
  - "Phase 2 target registration: YELLOW — adds targets to receive traffic"
  - "Phase 2 deregistration delay changes: YELLOW — affects in-flight request handling"

## Escalation Conditions

escalation_conditions:
  - "No targets can be registered in a production target group"
  - "Deregistration is causing dropped connections in production"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Auto Scaling integration is broken affecting capacity management"
  - "Cross-account or cross-VPC registration is required"

## Data Sensitivity

data_sensitivity:
  - command: "describe-target-health"
    sensitivity: LOW
    contains: "Target IDs and health status"
  - command: "cloudtrail lookup-events"
    sensitivity: HIGH
    contains: "API call details including principal ARNs"
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Instance details and VPC configuration"

## Prohibited Actions

prohibited_actions:
  - "NEVER deregister all targets from a production target group simultaneously"
  - "NEVER set deregistration delay to 0 in production without understanding the impact"
  - "NEVER register targets in the wrong VPC to work around connectivity issues"
  - "NEVER bypass Auto Scaling by manually registering targets in ASG-managed groups"

## Phase 3 — Rollback

- Before deregistering targets, document current registrations: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- If newly registered targets cause issues, deregister them: `aws elbv2 deregister-targets --target-group-arn <tg-arn> --targets Id=<target-id>`
- If deregistration delay was changed, restore the previous value
- If ASG target group association was modified, verify it matches the intended configuration
- Monitor HealthyHostCount after any registration changes
