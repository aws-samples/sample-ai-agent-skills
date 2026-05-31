# IAM Access Analyzer Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Access Analyzer issue.

## Guardrail 1: Two Analyzer Types with Different Scopes
ACCOUNT analyzers analyze resources within a single account. ORGANIZATION analyzers analyze across all accounts in the organization. Organization analyzers require the management account or delegated admin. Do not confuse the two types.

## Guardrail 2: External Access and Unused Access Are Separate
External access analysis (finding resources shared externally) and unused access analysis (finding unused IAM entities) are separate capabilities with separate pricing. They use different analyzer configurations and produce different finding types.

## Guardrail 3: Findings Are Asynchronous
Access Analyzer generates findings asynchronously. New resources or policy changes may take up to 24 hours to produce findings. Do not assume the absence of findings means no issues exist — analysis may not have completed yet.

## Guardrail 4: Policy Validation Checks Grammar Not Effectiveness
Policy validation (validate-policy) checks IAM policy syntax, grammar, and best practices. It does NOT validate whether the policy grants the intended permissions or is effective for the use case. It catches structural issues, not logical ones.

## Guardrail 5: Policy Generation Requires CloudTrail History
Policy generation creates least-privilege policies based on CloudTrail activity. It requires at least 90 days of CloudTrail logging for accurate results. Short activity windows produce incomplete policies that may miss required permissions.

## Guardrail 6: Archive Rules Hide But Don't Delete Findings
Archive rules automatically archive findings matching specified criteria. Archived findings are NOT deleted — they move to archived status and can be unarchived. Archive rules apply to future findings and can optionally apply to existing findings.

## Guardrail 7: Supported Resource Types Are Limited
External access analysis supports specific resource types: S3 buckets, IAM roles, KMS keys, Lambda functions, SQS queues, Secrets Manager secrets, SNS topics, EBS snapshots, RDS snapshots, ECR repositories, and EFS file systems. Other resource types are not analyzed.

## Guardrail 8: Organization Analyzer Requires Delegated Admin
Organization-level analyzers require either the management account or a delegated administrator account. Member accounts cannot create organization analyzers. Verify admin status: aws organizations list-delegated-administrators.

## Guardrail 9: Findings Include Condition Context
Findings include condition context showing under what conditions external access is granted. A finding with conditions may not represent unconditional public access. Always review the condition field before concluding on risk level.

## Guardrail 10: Unused Access Has Separate Pricing
Unused access analysis charges per IAM role or user analyzed per month. This is separate from external access analysis pricing. Large organizations with many IAM entities can incur significant costs.

## Guardrail 11: Custom Policy Checks Validate Against Requirements
Custom policy checks allow validating policies against your specific requirements (e.g., "no S3 public access"). This is different from standard validation. Custom checks use reference policies for comparison.

## Guardrail 12: Access Analyzer Is Regional for External Access
External access analyzers are regional — they analyze resources in the region where the analyzer is created. Create analyzers in each region with resources. Unused access analyzers are global (analyze IAM which is global).
