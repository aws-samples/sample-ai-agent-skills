# EC2 Auto Scaling Diagnostics Skill

Agent skill for investigating and troubleshooting EC2 Auto Scaling problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for EC2 Auto Scaling when the console alone isn't enough — scaling activities, launch failures, health check configuration, scaling policies, lifecycle hooks, termination policies, ELB integration, EventBridge events, mixed instances policies, spot allocation, warm pools, instance refresh, and predictive scaling.

### Activate When

- Auto Scaling group not scaling out under load
- Group not scaling in when load decreases
- Scaling too aggressively or oscillating
- Cooldown period causing missed scaling events
- Instance launch failures in scaling activities
- Launch template or launch configuration errors
- AMI not found or AMI permission issues
- Health check failures terminating healthy instances
- Instance refresh stuck or failing
- Warm pool not pre-warming instances
- Target tracking policy not maintaining target
- Step scaling policy not triggering correctly
- Predictive scaling forecasts inaccurate
- Lifecycle hooks timing out
- Unexpected termination policy behavior
- ELB health check integration issues
- EventBridge not receiving Auto Scaling events
- Mixed instances policy allocation failures
- Spot capacity unavailable
- General Auto Scaling errors without clear symptoms

---

## Skill Structure

```
autoscaling-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-not-scaling-out.md
    ├── A2-not-scaling-in.md
    ├── A3-scaling-too-aggressively.md
    ├── A4-cooldown-issues.md
    ├── B1-launch-failures.md
    ├── B2-launch-template-errors.md
    ├── B3-ami-issues.md
    ├── C1-health-check-failures.md
    ├── C2-instance-refresh.md
    ├── C3-warm-pool.md
    ├── D1-target-tracking.md
    ├── D2-step-scaling.md
    ├── D3-predictive-scaling.md
    ├── E1-lifecycle-hooks.md
    ├── E2-termination-policies.md
    ├── F1-elb-integration.md
    ├── F2-eventbridge.md
    ├── G1-mixed-instances-policy.md
    ├── G2-spot-allocation.md
    ├── Z1-general-troubleshooting.md
    ├── autoscaling-guardrails.md
    └── autoscaling-hallucination-patterns.yaml
```

---

## Runbook Library (26 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Scaling** | A1–A4 | Not scaling out, not scaling in, scaling too aggressively, cooldown issues |
| **B — Launch** | B1–B3 | Launch failures, launch template errors, AMI issues |
| **C — Health** | C1–C3 | Health check failures, instance refresh, warm pool |
| **D — Policies** | D1–D3 | Target tracking, step scaling, predictive scaling |
| **E — Lifecycle** | E1–E2 | Lifecycle hooks, termination policies |
| **F — Integration** | F1–F2 | ELB integration, EventBridge |
| **G — Mixed Instances** | G1–G2 | Mixed instances policy, spot allocation |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/autoscaling-guardrails.md` covering health check grace period vs cooldown distinction, ELB health check explicit enablement, scaling activity failure diagnosis, cooldown behavior across policy types, desired capacity conflict resolution, launch template vs launch configuration differences, mixed instances allocation strategies, warm pool lifecycle states, instance refresh MinHealthyPercentage, predictive scaling data requirements, lifecycle hook timeout behavior, and termination policy evaluation order.

---

## Investigation Workflow

1. **Triage** — Check ASG configuration, recent scaling activities, current capacity vs desired
2. **Deep Dive** — Examine scaling policies, launch template, health check settings, lifecycle hooks
3. **Detailed** — Review CloudTrail events, CloudWatch alarms, target group health, EventBridge rules

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `autoscaling:Describe*`, `ec2:DescribeLaunchTemplate*`, `ec2:DescribeInstances`, `elasticloadbalancing:DescribeTargetHealth`, `cloudwatch:DescribeAlarms`, `cloudtrail:LookupEvents`
- CloudWatch metrics enabled for the Auto Scaling group
- Access to target group health data for ELB-integrated ASGs

---

## Usage Examples

```
# Check ASG status and configuration
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names my-asg

# Review recent scaling activities
aws autoscaling describe-scaling-activities --auto-scaling-group-name my-asg --max-items 10

# Check scaling policies
aws autoscaling describe-policies --auto-scaling-group-name my-asg

# Check instance health in target group
aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:...

# Check instance refresh status
aws autoscaling describe-instance-refreshes --auto-scaling-group-name my-asg
```

---

## License

MIT-0
