---
name: computeoptimizer-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Compute Optimizer
  problems by analyzing EC2 recommendations, Lambda recommendations,
  EBS recommendations, ECS recommendations, enrollment, export, and
  following structured runbooks. Activate when: missing recommendations,
  wrong recommendations, enrollment issues, export failures, or the
  user says something is wrong with Compute Optimizer.
compatibility: >
  Requires AWS CLI with compute-optimizer, EC2, Lambda, ECS, CloudWatch,
  S3, and IAM permissions.
---

# AWS Compute Optimizer Diagnostics

## When to use

Any AWS Compute Optimizer investigation — EC2/Lambda/EBS/ECS recommendations, enrollment, or export.

## Investigation workflow

### Step 1 — Collect and triage

```
aws compute-optimizer get-enrollment-status
aws compute-optimizer get-enrollment-statuses-for-organization
aws compute-optimizer get-recommendation-summaries
```

### Step 2 — Domain deep dive

```
aws compute-optimizer get-ec2-instance-recommendations --instance-arns <arn>
aws compute-optimizer get-lambda-function-recommendations --function-arns <arn>
aws compute-optimizer get-ebs-volume-recommendations --volume-arns <arn>
aws compute-optimizer get-ecs-service-recommendations --service-arns <arn>
```

### Step 3 — Detailed investigation

```
aws compute-optimizer get-ec2-recommendation-projected-metrics --instance-arn <arn> --stat Maximum --period 86400 --start-time <start> --end-time <end>
aws compute-optimizer describe-recommendation-export-jobs
```

Read `references/guardrails.md` before concluding on any Compute Optimizer issue.

## Gotchas: AWS Compute Optimizer

- Account must be enrolled (opted in) before recommendations appear.
- Recommendations require 14+ days of CloudWatch metric data.
- Enhanced infrastructure metrics (paid) provide 3-month lookback.
- Export jobs write to S3 and are asynchronous.
- Organization enrollment requires management account action.
- Recommendations are resource-type specific with different criteria.

## Anti-hallucination rules

1. Always cite resource ARNs, recommendation details, or API responses as evidence.
2. Enrollment is required. Never assume recommendations exist without opt-in.
3. 14 days of data minimum. Never expect instant recommendations.
4. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — EC2 | A1-A2 | Instance recommendations, projected metrics |
| B — Lambda | B1-B2 | Function recommendations, memory sizing |
| C — EBS | C1-C2 | Volume recommendations, IOPS optimization |
| D — ECS | D1-D2 | Service recommendations, task sizing |
| E — Enrollment | E1-E2 | Account enrollment, organization enrollment |
| F — Export | F1-F2 | Export jobs, export configuration |
