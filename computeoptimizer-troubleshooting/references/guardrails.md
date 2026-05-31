# AWS Compute Optimizer Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Compute Optimizer issue.

## Guardrail 1: Account Must Be Enrolled
Compute Optimizer requires explicit enrollment (opt-in). Without enrollment, no recommendations are generated. Check enrollment status with get-enrollment-status.

## Guardrail 2: Recommendations Require 14+ Days of Metric Data
Resources must have at least 14 days of CloudWatch metric data before recommendations appear. New resources have no recommendations until sufficient data accumulates.

## Guardrail 3: Enhanced Infrastructure Metrics Is Paid
The default lookback period is 14 days. Enhanced infrastructure metrics extends this to 3 months but incurs additional charges. Better recommendations with longer lookback.

## Guardrail 4: Recommendations Are Not Prescriptive
Recommendations suggest options with projected performance. They are not guarantees. Workload testing after changes is essential.

## Guardrail 5: Organization Enrollment Requires Management Account
Enrolling all accounts in an organization requires action from the management account or delegated administrator.

## Guardrail 6: Export Jobs Are Asynchronous
Recommendation exports write to S3 and take time. Export jobs have status tracking. S3 bucket must be accessible.

## Guardrail 7: EC2 Recommendations Consider Multiple Metrics
EC2 recommendations analyze CPU, memory (if CloudWatch agent installed), network, and disk. Without CloudWatch agent, memory data is missing.

## Guardrail 8: Lambda Recommendations Focus on Memory
Lambda recommendations primarily optimize memory allocation. They analyze invocation patterns, duration, and memory utilization.

## Guardrail 9: EBS Recommendations Consider IOPS and Throughput
EBS recommendations analyze volume IOPS, throughput, and size utilization. They may suggest volume type changes.

## Guardrail 10: ECS Recommendations Analyze Task-Level Metrics
ECS recommendations analyze CPU and memory at the task level. They require Container Insights enabled.

## Guardrail 11: Finding Classifications Indicate Optimization State
Resources are classified as Over-provisioned, Under-provisioned, Optimized, or Not Optimized. Each has different recommendation implications.

## Guardrail 12: Savings Estimates Are Approximate
Cost savings estimates are based on On-Demand pricing. Actual savings depend on pricing model (Reserved, Savings Plans, Spot).
