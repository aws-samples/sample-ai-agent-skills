# Trusted Advisor Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Trusted Advisor issue.

## Guardrail 1: API Requires Business or Enterprise Support
The full Trusted Advisor API (support:describeTrustedAdvisor*) requires Business or Enterprise Support plan. Basic and Developer plans only have access to 7 core checks (S3 bucket permissions, security groups, IAM use, MFA on root, EBS snapshots, RDS snapshots, service limits). Never suggest full API access for Basic/Developer plans.

## Guardrail 2: API Endpoint Is us-east-1 Only
The Trusted Advisor API is only available through the us-east-1 (N. Virginia) endpoint. All support API calls must specify --region us-east-1. Calls to other regions return errors. This applies to all Trusted Advisor API operations.

## Guardrail 3: Check Results May Be Stale
Trusted Advisor check results are point-in-time snapshots from the last refresh. They are not real-time. Some checks auto-refresh weekly, others require manual refresh. Always check the last refresh timestamp before acting on results. Stale results may not reflect current resource state.

## Guardrail 4: Minimum 5-Minute Refresh Interval
Trusted Advisor checks have a minimum refresh interval of 5 minutes. Requesting a refresh before the interval expires returns an InvalidParameterValueException. Do not attempt rapid successive refreshes. Check refresh status before requesting a new refresh.

## Guardrail 5: Some Checks Are Regional Others Are Global
Some Trusted Advisor checks evaluate resources globally (IAM, Route 53), while others are region-specific (EC2, RDS). A green status on a global check means all regions are clean. A green status on a regional check only covers the evaluated region.

## Guardrail 6: Priority Requires Enterprise Support with TAM
Trusted Advisor Priority is only available with Enterprise Support plan and requires a Technical Account Manager (TAM). Priority provides curated, prioritized recommendations. Never suggest Priority features for Business or lower support plans.

## Guardrail 7: Organizational View Requires All Features Enabled
Trusted Advisor organizational view requires AWS Organizations with all features enabled (not just consolidated billing). Trusted Advisor must be enabled as a trusted service in Organizations. The management account or delegated administrator can access organizational view.

## Guardrail 8: Excluding Resources Hides Not Fixes
Excluding resources from Trusted Advisor checks only hides them from the dashboard and summary counts. It does not fix the underlying issue or suppress the finding permanently. Excluded resources can be re-included at any time. Never suggest excluding as a remediation.

## Guardrail 9: Service Limit Checks May Lag Quota Increases
Trusted Advisor service limit checks may not immediately reflect recent Service Quotas increases. There can be a delay of hours to days before the updated limit appears. Do not diagnose a service limit warning as incorrect without checking the actual quota.

## Guardrail 10: EventBridge Publishes Status Changes Not Details
Trusted Advisor EventBridge integration publishes check status changes (ok, warning, error) not individual resource-level findings. To get resource details, you must call the API after receiving the status change event.

## Guardrail 11: Free Checks Available on All Plans
Seven core Trusted Advisor checks are available on all support plans including Basic: S3 Bucket Permissions, Security Groups (unrestricted access), IAM Use, MFA on Root Account, EBS Public Snapshots, RDS Public Snapshots, and Service Limits. These do not require Business/Enterprise support.

## Guardrail 12: Check IDs Are Stable but Check Names May Change
Trusted Advisor check IDs are stable identifiers that do not change. However, check names and descriptions may be updated by AWS. Always reference checks by ID in automation, not by name. Use describe-trusted-advisor-checks to get current names.
