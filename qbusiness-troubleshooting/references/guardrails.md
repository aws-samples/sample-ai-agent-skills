# Amazon Q Business Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Q Business issue.

## Guardrail 1: IAM Identity Center Is Mandatory
Q Business applications require AWS IAM Identity Center (SSO) for user authentication. Without Identity Center configured, application creation fails. Users must be provisioned in Identity Center before they can access Q Business.

## Guardrail 2: Data Source Sync Is Asynchronous
After configuring a data source connector, sync must be triggered. Documents are crawled, processed, chunked, and indexed. This takes minutes to hours. Chat responses only include synced documents.

## Guardrail 3: Topic Controls Can Block Legitimate Queries
Q Business guardrails (topic controls) define allowed and blocked topics. Overly restrictive controls block legitimate business queries. Test controls with representative queries before production.

## Guardrail 4: Plugins Require OAuth or Service Configuration
Q Business plugins connect to external services (Jira, Salesforce, etc.). Each plugin requires proper authentication configuration. Plugin failures affect the conversation flow and may return errors to users.

## Guardrail 5: Web Experience Requires User Assignment
The web experience URL is only accessible to users assigned in IAM Identity Center. Unassigned users get access denied. User roles (admin, user) control capabilities.

## Guardrail 6: Retrieval Quality Depends on Document Chunking
Q Business chunks documents for retrieval. Chunk size and overlap affect answer quality. Poor chunking leads to incomplete or irrelevant answers. Custom chunking strategies may be needed.

## Guardrail 7: Application Deletion Is Irreversible
Deleting a Q Business application removes all indices, data sources, and configurations. There is no undo. Ensure backups of configurations before deletion.

## Guardrail 8: Cross-Account Data Sources Require Additional Setup
Accessing data sources in other AWS accounts requires cross-account IAM roles and proper trust policies. VPC connectivity may also be needed.

## Guardrail 9: Response Controls Affect All Users
Global response controls (blocked topics, response length) apply to all users. Per-user customization requires different application configurations or topic control rules.

## Guardrail 10: Document ACLs Are Inherited from Data Sources
Q Business respects document-level ACLs from data sources. Users only see documents they have access to. Misconfigured ACLs cause users to get no answers.

## Guardrail 11: Plugin Actions Require User Consent
When Q Business invokes a plugin action, it may require user confirmation. Plugin timeout settings affect user experience. Failed plugins should gracefully degrade.

## Guardrail 12: Index Capacity Affects Performance
Q Business indices have document and storage limits. Exceeding limits degrades performance or blocks ingestion. Monitor index utilization.
