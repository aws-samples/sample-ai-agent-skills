---
name: autoscaling-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot EC2 Auto Scaling problems
  by analyzing scaling activities, launch failures, health checks, scaling
  policies, lifecycle hooks, ELB integration, and mixed instances configuration
  using structured runbooks. Activate when: group not scaling out, not scaling in,
  scaling too aggressively, cooldown issues, launch failures, launch template
  errors, AMI issues, health check failures, instance refresh problems, warm pool
  issues, target tracking not working, step scaling misconfigured, predictive
  scaling inaccurate, lifecycle hook timeouts, termination policy unexpected,
  ELB health check mismatches, EventBridge events missing, mixed instances
  allocation failures, spot capacity issues, or the user says something is
  wrong with Auto Scaling without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with AutoScaling, EC2, ELB, CloudWatch,
  CloudTrail, and optionally EventBridge permissions.
---

# EC2 Auto Scaling Diagnostics

## When to use

Any EC2 Auto Scaling investigation where the console alone is insufficient — scaling not triggering, launch failures, health check mismatches, policy tuning, lifecycle hook issues, ELB integration problems, mixed instances configuration, or warm pool troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name>
aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name> --max-items 20
aws autoscaling describe-auto-scaling-instances --instance-ids <instance-id>
aws cloudwatch get-metric-statistics --namespace AWS/AutoScaling --metric-name GroupDesiredCapacity --dimensions Name=AutoScalingGroupName,Value=<asg-name> --start-time <start> --end-time <end> --period 300 --statistics Average
```

### Step 2 — Domain deep dive

```
aws autoscaling describe-policies --auto-scaling-group-name <asg-name>
aws autoscaling describe-launch-configurations --launch-configuration-names <lc-name>
aws ec2 describe-launch-template-versions --launch-template-id <lt-id>
aws autoscaling describe-lifecycle-hooks --auto-scaling-group-name <asg-name>
aws autoscaling describe-warm-pool --auto-scaling-group-name <asg-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=autoscaling.amazonaws.com --max-results 20
aws autoscaling describe-notification-configurations --auto-scaling-group-names <asg-name>
aws elbv2 describe-target-health --target-group-arn <tg-arn>
aws cloudwatch describe-alarms --alarm-name-prefix <asg-alarm-prefix>
```

Read `references/autoscaling-guardrails.md` before concluding on any Auto Scaling issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-auto-scaling-groups` | Check ASG config, desired/min/max, health check type |
| `describe-scaling-activities` | Review recent scaling events and failures |
| `describe-policies` | Check scaling policy configuration |
| `describe-launch-configurations` | Check legacy launch config settings |
| `describe-launch-template-versions` | Check launch template settings |
| `describe-lifecycle-hooks` | Check lifecycle hook configuration |
| `describe-warm-pool` | Check warm pool status and config |
| `describe-target-health` | Check instance health in target groups |
| `describe-instance-refreshes` | Check instance refresh status |
| CloudWatch Alarms | Check scaling alarm states |

## Gotchas: EC2 Auto Scaling

- Health check grace period delays health evaluation. New instances are not checked until the grace period expires. If the grace period is too short, instances may be terminated before the application starts. If too long, unhealthy instances remain in service.
- Cooldown periods prevent rapid scaling. Default cooldown is 300 seconds. During cooldown, Auto Scaling ignores additional scaling triggers. Target tracking policies have their own cooldown. Step scaling policies can override the default cooldown.
- ELB health checks are separate from EC2 health checks. By default, ASGs use EC2 status checks only. ELB health checks must be explicitly enabled. An instance can pass EC2 checks but fail ELB checks. Both must pass when ELB health checks are enabled.
- Scaling activities can fail silently. Check describe-scaling-activities for StatusCode=Failed. Common causes: IAM permissions, subnet capacity, instance type unavailable, launch template errors.
- Mixed instances policies use allocation strategies. Spot instances use lowest-price or capacity-optimized. On-Demand uses prioritized or lowest-price. The base capacity determines minimum On-Demand instances.
- Instance refresh replaces instances gradually. It respects MinHealthyPercentage. A failed health check during refresh can pause the operation. Rollback is available but not automatic by default.
- Warm pool instances are pre-initialized. They can be in Stopped, Running, or Hibernated state. Warm pool has its own max size. Lifecycle hooks apply to warm pool transitions.
- Predictive scaling uses ML forecasting. It requires 24 hours of data minimum. Forecasts are generated daily. It works best with recurring patterns. It can be combined with dynamic scaling.
- Desired capacity can be modified by multiple sources. Scaling policies, scheduled actions, manual updates, and external tools can all change desired capacity. Conflicts can cause oscillation.

## Anti-hallucination rules

1. Always cite specific scaling activity IDs or API responses as evidence.
2. Health check grace period is NOT the same as cooldown period. Never confuse the two.
3. ELB health checks must be explicitly enabled on the ASG. Never assume they are active.
4. Cooldown applies to the ASG, not individual policies (except target tracking).
5. Mixed instances policies require specific instance type flexibility. Never assume any type works.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 26 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Scaling | A1-A4 | Not scaling out, not scaling in, scaling too aggressively, cooldown issues |
| B — Launch | B1-B3 | Launch failures, launch template errors, AMI issues |
| C — Health | C1-C3 | Health check failures, instance refresh, warm pool |
| D — Policies | D1-D3 | Target tracking, step scaling, predictive scaling |
| E — Lifecycle | E1-E2 | Lifecycle hooks, termination policies |
| F — Integration | F1-F2 | ELB integration, EventBridge |
| G — Mixed Instances | G1-G2 | Mixed instances policy, spot allocation |
| Z — Catch-All | Z1 | General troubleshooting |
