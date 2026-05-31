# Amazon DevOps Guru Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon DevOps Guru issue.

## Guardrail 1: DevOps Guru Requires Learning Period
After enabling resource coverage, DevOps Guru needs hours to days to learn normal behavior patterns. Anomalies are not detected immediately. The ML models build baselines from CloudWatch metrics and operational data.

## Guardrail 2: Resource Coverage Must Be Explicitly Configured
Resources are not automatically monitored. Coverage must be configured via CloudFormation stacks, AWS tags, or account-wide setting. Uncovered resources generate no insights.

## Guardrail 3: Insights Combine Multiple Anomalies
A single insight may contain multiple related anomalies across different metrics and resources. Insights represent operational issues, not individual metric deviations.

## Guardrail 4: Proactive Insights Require Historical Data
Proactive insights predict future issues based on historical patterns. They require weeks of data to generate meaningful predictions. New resources have no proactive insights.

## Guardrail 5: Cost Estimation Is Approximate
The cost estimation API provides approximate monthly costs based on monitored resources. Actual costs depend on resource count, metric volume, and API call frequency.

## Guardrail 6: SNS Notifications Must Be Configured
DevOps Guru does not send notifications by default. SNS topics must be created and configured as notification channels. Each channel can filter by severity.

## Guardrail 7: CloudFormation Stack Coverage Has Limits
There are limits on the number of CloudFormation stacks that can be monitored. Large accounts may need to use tag-based coverage instead.

## Guardrail 8: Anomaly Severity Affects Insight Priority
Anomalies have severity levels (HIGH, MEDIUM, LOW). Insight severity is derived from the highest-severity anomaly. Focus on HIGH severity insights first.

## Guardrail 9: Organization-Level Insights Require Delegated Admin
Multi-account insights require AWS Organizations integration with a delegated administrator account. Member accounts must be enrolled.

## Guardrail 10: Recommendations Are Contextual
Insight recommendations are based on the specific anomalies detected. They are not generic best practices. Follow recommendations in the context of the specific insight.

## Guardrail 11: Metric Anomalies Use Statistical Models
Anomaly detection uses statistical models, not simple thresholds. What constitutes an anomaly depends on historical patterns. Seasonal variations are accounted for.

## Guardrail 12: Closing Insights Does Not Prevent Recurrence
Closing an insight marks it as resolved but does not prevent the same issue from generating a new insight. Address the root cause to prevent recurrence.
