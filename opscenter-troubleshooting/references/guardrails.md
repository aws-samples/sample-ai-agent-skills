# AWS SSM OpsCenter Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any OpsCenter issue.

## Guardrail 1: OpsCenter Is Not Incident Management
OpsCenter tracks operational issues (OpsItems), not incidents. For incident management, use AWS Incident Manager. OpsItems are for tracking, investigating, and resolving operational issues. Do not conflate OpsCenter with incident response.

## Guardrail 2: OpsCenter Requires SSM Setup
OpsCenter is part of Systems Manager. SSM must be properly configured in the account and region. Verify OpsCenter is enabled: aws ssm get-service-setting --setting-id /ssm/opsCenter.

## Guardrail 3: Automation Runbooks Need IAM Roles
Runbooks associated with OpsItems execute as SSM Automation documents. They require an IAM role (AutomationAssumeRole) with permissions for all actions in the runbook. Missing permissions cause execution failures.

## Guardrail 4: Related Resources Are Static References
Related resources attached to OpsItems are ARN references. They do NOT auto-update when the resource state changes. If a related EC2 instance is terminated, the OpsItem still shows the ARN. Always verify current resource state separately.

## Guardrail 5: Deduplication Uses Operational Data Hash
OpsItem deduplication is based on the dedup string in operational data. OpsItems with the same dedup string are merged into a single OpsItem. Without a dedup string, every creation results in a new OpsItem.

## Guardrail 6: EventBridge Integration Requires Rules
AWS services create OpsItems through EventBridge rules. If the EventBridge rule is disabled or misconfigured, OpsItems won't be created automatically. Verify rules: aws events list-rules --name-prefix "SSM".

## Guardrail 7: OpsItem Severity Is Informational
OpsItem severity (1-4) is for prioritization and filtering. It does NOT trigger automatic actions or escalations. Severity-based automation must be configured separately through EventBridge rules or Lambda functions.

## Guardrail 8: OpsItem Status Workflow
OpsItems follow a status workflow: Open → InProgress → Resolved (or other custom statuses). Status changes are manual or automation-driven. There is no automatic status progression.

## Guardrail 9: Cross-Account OpsItems Need Organizations
Viewing OpsItems across accounts requires AWS Organizations and SSM Explorer/OpsData configuration. Individual account OpsItems are not visible from other accounts without this setup.

## Guardrail 10: Notifications Require SNS Configuration
OpsItem notifications are not built-in. You must configure EventBridge rules to match OpsItem events and route to SNS topics for notifications. There is no native OpsCenter notification mechanism.

## Guardrail 11: OpsCenter Is Regional
OpsItems are stored regionally. An OpsItem created in us-east-1 is not visible in eu-west-1. Use SSM Explorer for cross-region aggregation and visibility.

## Guardrail 12: Source Field Tracks Origin
The Source field in OpsItems identifies what created the OpsItem (aws/config, aws/cloudwatch, manual, etc.). Use this field for filtering and understanding OpsItem origin. Do not modify the Source field as it breaks tracking.
