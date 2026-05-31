# AWS Security Hub Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Security Hub issue.

## Guardrail 1: Security Hub Requires Config for Most Controls
Most Security Hub security controls evaluate AWS Config rules under the hood. Without Config recorder running and recording the required resource types, controls show NO_DATA or UNKNOWN status. Before troubleshooting any control failure, always verify Config is enabled with `describe-configuration-recorder-status`. This is the single most common root cause of Security Hub control issues.

## Guardrail 2: Controls Are Backed by Config Rules — Not Independent
Security Hub controls are not independent security checks. They evaluate underlying AWS Config rules. If the Config rule is not evaluating (recorder stopped, rule deleted, Lambda error for custom rules), the Security Hub control will not produce results. Always check the corresponding Config rule when a control shows unexpected results.

## Guardrail 3: Finding Deduplication Uses ProductArn + Id
Each finding in Security Hub is uniquely identified by the combination of ProductArn and Id fields. When a finding with the same ProductArn and Id is imported again, it updates the existing finding rather than creating a duplicate. Different products can have findings with the same Id without conflict because the ProductArn differs. Never assume finding uniqueness based on Id alone.

## Guardrail 4: ASFF Is the Required Format for All Findings
All findings in Security Hub must conform to the AWS Security Finding Format (ASFF). Third-party integrations must transform their findings to ASFF before import. Required ASFF fields include: SchemaVersion, Id, ProductArn, GeneratorId, AwsAccountId, Types, CreatedAt, UpdatedAt, Severity, Title, and Resources. Missing or invalid required fields cause import failures with BatchImportFindings.

## Guardrail 5: Cross-Region Aggregation Requires Explicit Setup
Findings from other regions are NOT automatically visible in any single region. A finding aggregator must be explicitly created in the aggregation region using `create-finding-aggregator`. The aggregator can link specific regions or all regions. Cross-region aggregation is separate from cross-account aggregation. Both must be configured independently for full centralization.

## Guardrail 6: Compliance Score Is Percentage of Passed Controls
The security score for each standard is calculated as: (number of passed controls / total number of enabled controls) × 100. Disabled controls are excluded from both numerator and denominator. Controls with NO_DATA or NOT_AVAILABLE status count as NOT passed (reduce the score). Disabling failing controls artificially improves the score but does not improve security.

## Guardrail 7: Disabling a Control Does Not Delete Existing Findings
When a security control is disabled, it stops generating NEW findings and stops evaluating. However, existing findings from that control remain in Security Hub until they are manually archived or reach the 90-day retention limit. Re-enabling a control triggers fresh evaluation and may generate new findings. Disabled controls are excluded from compliance score calculation.

## Guardrail 8: Custom Actions Require EventBridge Rules to Be Useful
Custom actions in Security Hub send events to EventBridge when triggered (manually or via automation). Without corresponding EventBridge rules to process these events, custom actions do nothing. The EventBridge event source is `aws.securityhub` with detail-type `Security Hub Findings - Custom Action`. Each custom action has a unique ARN used in the EventBridge event pattern.

## Guardrail 9: Pricing Is Based on Security Checks and Finding Ingestion
Security Hub pricing has two components: security checks (control evaluations) and finding ingestion events. Each control evaluation per resource counts as a security check. Each finding imported from integrated products counts as a finding ingestion event. Enabling many standards with many controls across many resources increases costs significantly. Cross-region aggregation does not incur additional finding charges.

## Guardrail 10: Standard Enablement Is Per-Region and Per-Account
Security standards must be enabled in each region and each account separately. Enabling a standard in one region does not enable it in others. In multi-account setups, the delegated admin can enable standards for member accounts, but this is still per-region. Organization-level configuration can auto-enable standards for new member accounts.

## Guardrail 11: Finding Retention Is 90 Days After Last Update
Security Hub retains findings for 90 days after the last update (UpdatedAt timestamp). Findings not updated within 90 days are automatically deleted. Active findings that are continuously updated by their source product are retained indefinitely. For long-term retention, export findings to S3 using EventBridge rules or the Security Hub API.

## Guardrail 12: Consolidated Control Findings Changes Behavior
When consolidated control findings is enabled (default for new accounts), Security Hub generates a single finding per security check per resource, regardless of how many standards the control is associated with. Without consolidation, separate findings are generated for each standard. This affects finding count, deduplication, and compliance reporting.
