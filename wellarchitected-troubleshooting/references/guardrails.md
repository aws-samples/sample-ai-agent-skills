# AWS Well-Architected Tool Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Well-Architected Tool issue.

## Guardrail 1: Reviews Are Manual Question-Based Assessments
The Well-Architected Tool does NOT automatically scan infrastructure. Users answer questions about their workload design. Answers drive improvement plans.

## Guardrail 2: Multiple Lenses Are Available
Beyond the main Well-Architected Framework, specialized lenses exist for Serverless, SaaS, ML, Data Analytics, and more. Select appropriate lenses for the workload type.

## Guardrail 3: Milestones Are Immutable Snapshots
Once created, milestones cannot be modified. They capture the review state at a point in time. Create new milestones to track progress.

## Guardrail 4: Custom Lenses Follow Strict JSON Schema
Custom lens definitions must follow the Well-Architected custom lens JSON schema. Schema validation is strict. Invalid schemas cause import failures.

## Guardrail 5: Sharing Requires Invitation Acceptance
Workload sharing sends invitations that must be accepted. Invitations expire after 7 days. Sharing can be read-only or contributor access.

## Guardrail 6: Improvement Plans Are Generated From Answers
Improvement plans list high-risk and medium-risk items based on review answers. They provide best practice references but do not implement changes.

## Guardrail 7: Workload Definitions Affect Review Scope
Workload properties (environment, regions, account IDs) provide context for the review. Incorrect workload definitions may lead to irrelevant questions.

## Guardrail 8: Lens Versions May Change Questions
AWS official lenses are updated periodically. New versions may add, modify, or remove questions. Upgrading lens version on a workload may require re-answering questions.

## Guardrail 9: Risk Counts Drive Improvement Priority
Workloads show High Risk Issue (HRI) and Medium Risk Issue (MRI) counts. These counts drive improvement prioritization. Not all questions contribute equally to risk.

## Guardrail 10: Cross-Account Sharing Uses AWS Organizations
Organization-level sharing leverages AWS Organizations. Sharing outside the organization uses account ID-based invitations.

## Guardrail 11: Custom Lens Publishing Is Separate From Import
Importing a custom lens makes it available in your account. Publishing makes it available to other accounts. These are separate operations.

## Guardrail 12: Workload Deletion Is Permanent
Deleting a workload removes all reviews, milestones, and sharing. There is no undo. Export data before deletion.
