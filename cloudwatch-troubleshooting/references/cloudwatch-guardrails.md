# CloudWatch Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any CloudWatch issue.

## Guardrail 1: Log Group Retention Defaults to Never Expire
Log group retention is infinite by default. Logs are stored forever and costs accumulate indefinitely. Always check retention settings and recommend explicit retention policies. Never assume logs are automatically deleted.

## Guardrail 2: INSUFFICIENT_DATA Is Not an Error
INSUFFICIENT_DATA means no metric data was received during the evaluation period. It does not mean the monitored resource is down or broken. The resource may not be emitting data (e.g., Lambda function not invoked, EC2 instance stopped). Always verify metric publishing before diagnosing.

## Guardrail 3: Metric Resolution Affects Availability
Standard metrics have 1-minute minimum period. High-resolution custom metrics support 1-second periods. Requesting a period shorter than the metric's resolution returns no data. Standard AWS service metrics cannot be queried at sub-minute granularity.

## Guardrail 4: Alarm Evaluation Periods vs Datapoints to Alarm
An alarm with 3 evaluation periods and 2 datapoints to alarm requires 2 out of 3 consecutive periods to breach. Misunderstanding this M-of-N logic is the most common cause of "alarm not triggering" complaints. Always check both settings together.

## Guardrail 5: Composite Alarms Do Not Inherit Actions
Composite alarms aggregate child alarm states but do not automatically inherit or propagate actions from child alarms. Actions (SNS, Auto Scaling, EC2, Lambda) must be configured directly on the composite alarm. Never claim actions flow from child to composite.

## Guardrail 6: Metric Math Cannot Handle Missing Data
If any input metric in a metric math expression has missing data, the entire expression result is missing for that period. Treat Missing Data settings on the alarm do not fill in the metric math inputs. Never suggest metric math as a workaround for sparse metrics without addressing missing data.

## Guardrail 7: Cross-Account Requires Sink and Source Setup
Cross-account observability uses CloudWatch Observability Access Manager (OAM). The monitoring account creates a sink, and each source account creates a link to that sink. Both sides must be configured. A sink alone does nothing without linked source accounts.

## Guardrail 8: Synthetics Canaries Run on Lambda
Canaries execute on Lambda functions managed by the Synthetics service. They inherit Lambda limits: 15-minute max timeout, memory constraints, /tmp storage limits, and VPC cold start behavior. Never troubleshoot canary issues without considering Lambda constraints.

## Guardrail 9: Subscription Filters Have a Per-Log-Group Limit
Each log group supports a maximum of 2 subscription filters. Attempting to add more fails. If both slots are used, one must be removed before adding a new destination. Account-level subscription policies count toward this limit.

## Guardrail 10: Contributor Insights Charges Per Matched Event
Contributor Insights rules incur cost per matched log event, not per rule. A rule matching a high-volume log group can generate significant charges. Always estimate event volume before enabling rules on production log groups.

## Guardrail 11: Log Insights Queries Timeout at 15 Minutes
CloudWatch Logs Insights queries have a hard 15-minute timeout. Queries scanning large volumes of data across wide time ranges will timeout. Narrow the time range, add filters early in the query, or use the `limit` clause to reduce scan volume.

## Guardrail 12: CloudWatch Agent Requires IAM Permissions and Connectivity
The CloudWatch Agent needs IAM permissions (CloudWatchAgentServerPolicy) and network connectivity to CloudWatch endpoints. In VPCs without internet access, VPC endpoints for `monitoring` and `logs` are required. Missing either causes silent metric/log publishing failures.
