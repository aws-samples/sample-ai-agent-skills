# AWS Config Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Config issue.

## Guardrail 1: Config Recorder Must Be Running for Rules to Evaluate
The configuration recorder must be in a "recording" state for Config rules to receive configuration change events. If the recorder is stopped, change-triggered rules will never fire and periodic rules will not evaluate. Always verify recorder status with `describe-configuration-recorder-status` before investigating rule evaluation failures. A common mistake is assuming rules are broken when the recorder is simply stopped.

## Guardrail 2: Delivery Channel Is Required for Recording to Persist
A valid delivery channel (S3 bucket + optional SNS topic) must exist and be properly configured for the recorder to persist configuration items. If the S3 bucket is deleted, the bucket policy denies access, or the delivery channel is misconfigured, recording silently fails. The delivery channel status shows the last successful delivery time — a stale timestamp indicates delivery problems.

## Guardrail 3: Not All Resource Types Are Recorded by Default
When "Record all supported resource types" is enabled, it covers most AWS resource types but NOT all. Some resource types (especially newer services and third-party resources) require explicit inclusion. Global resources (IAM users, groups, roles, policies) are only recorded in the home region unless explicitly enabled in other regions. Recording global resources in multiple regions creates duplicate configuration items and increases costs.

## Guardrail 4: Custom Rules Are Lambda Functions with Lambda Limits
Custom Config rules invoke Lambda functions for evaluation. These functions are subject to all Lambda limits: 15-minute timeout, memory limits, concurrency limits, and cold starts. If the Lambda function is throttled, the rule evaluation fails with an error. The Lambda function must return evaluation results in the correct format — malformed responses cause silent failures. Check Lambda CloudWatch logs for custom rule debugging.

## Guardrail 5: Conformance Packs Are CloudFormation-Based with Limits
Conformance packs deploy as CloudFormation stacks. They are subject to CloudFormation limits: template size, resource count, and stack operation timeouts. A conformance pack can contain up to 50 Config rules. Organization conformance packs deploy to all member accounts and require the management account or delegated admin to manage. Failed deployments appear in CloudFormation stack events, not just Config console.

## Guardrail 6: Remediation Uses SSM Automation Documents
Auto-remediation executes SSM Automation runbooks (documents). The Config service role must have `ssm:StartAutomationExecution` permission and the specific permissions required by the automation document. Remediation failures appear in SSM Automation execution history — always check SSM console, not just Config. The remediation execution role is separate from the Config service role. Remediation has a maximum concurrency and error threshold.

## Guardrail 7: Aggregator Requires Explicit Authorization
Multi-account aggregators require explicit authorization from each source account using `put-aggregation-authorization`. Organization aggregators bypass this requirement but need Organizations access. Without proper authorization, the aggregator silently shows no data from unauthorized accounts. Cross-region aggregation within the same account does not require authorization. Aggregator data is eventually consistent — there is a delay.

## Guardrail 8: Advanced Queries Have Syntax and Scope Limitations
Advanced queries use a SQL-like syntax but only support a subset of SQL operations. Queries run against the CURRENT configuration snapshot only — not historical data. Not all resource attributes are queryable — only indexed properties work. Complex queries with many conditions may time out. The `SELECT *` syntax is not supported — you must specify fields. Query results are paginated with a maximum of 100 results per page.

## Guardrail 9: Organization Rules Deploy to All Member Accounts
Organization Config rules deploy to ALL member accounts in the organization (or specified OUs). They cannot be modified or deleted by member accounts — only the management account or delegated admin can manage them. If a member account does not have a recorder running, the organization rule will exist but not evaluate. Organization rules count toward the per-account rule limit in each member account.

## Guardrail 10: Config Costs Scale with Configuration Items and Evaluations
AWS Config charges per configuration item recorded and per rule evaluation. High-frequency changes on many resources (e.g., Auto Scaling groups, Lambda functions) can generate thousands of configuration items per day. Each rule evaluation against each resource is a separate charge. Use resource type filtering to record only needed types. Conformance packs with many rules multiply evaluation costs across all resources.

## Guardrail 11: Rule Evaluation Has Ordering and Timing Considerations
Change-triggered rules evaluate when a configuration item changes. Periodic rules evaluate on a schedule (1h, 3h, 6h, 12h, or 24h). There is no guaranteed ordering between rules. A resource may be evaluated by multiple rules simultaneously. Rule evaluation results are eventually consistent — there is a delay between the change and the compliance result appearing. Forcing a re-evaluation with `start-config-rules-evaluation` is asynchronous.

## Guardrail 12: Configuration Item Retention Has Limits
AWS Config retains configuration items for a configurable retention period (default 7 years, minimum 30 days, maximum 7 years). After the retention period, configuration items are automatically deleted. The `get-resource-config-history` API only returns items within the retention period. Configuration snapshots delivered to S3 are separate from the Config database and follow S3 lifecycle rules. Reducing retention does not retroactively delete existing items immediately.
