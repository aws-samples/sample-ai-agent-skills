# Clean Rooms Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Clean Rooms issue.

## Guardrail 1: Collaborations Require At Least Two Members
A Clean Rooms collaboration must have at least two member accounts. You cannot create a collaboration with a single account for testing. Use two separate AWS accounts. The creator account is automatically the first member.

## Guardrail 2: Analysis Rules Are Required for Querying
Every configured table must have an analysis rule (AGGREGATION, LIST, or CUSTOM) before it can be queried. Without an analysis rule, the table appears in the schema but cannot be used in queries. The analysis rule type determines what SQL operations are allowed.

## Guardrail 3: Configured Tables Require Glue Data Catalog
Tables must be registered in the AWS Glue Data Catalog before they can be configured in Clean Rooms. The Glue database and table must exist in the same account and region. Missing Glue catalog entries cause configured table creation to fail.

## Guardrail 4: Only CAN_RECEIVE_RESULTS Members Get Query Output
Only collaboration members with the CAN_RECEIVE_RESULTS ability can receive query results. The output S3 bucket must be in the same account as the receiving member. Other members can run queries but cannot see results.

## Guardrail 5: Raw Data Never Leaves the Owner's Account
Clean Rooms processes queries without moving raw data between accounts. Only aggregated, filtered, or otherwise processed results are shared. This is a fundamental privacy guarantee. Never suggest that raw data is copied or transferred.

## Guardrail 6: Differential Privacy Budget Depletes with Each Query
Differential privacy uses a finite privacy budget (epsilon). Each query consumes part of the budget. Once the budget is exhausted, no more queries can run until the budget is refreshed (typically monthly). Monitor budget consumption carefully.

## Guardrail 7: Cryptographic Computing Requires Pre-Encrypted Data
Clean Rooms cryptographic computing requires data to be encrypted client-side before upload to S3. It uses specific encryption schemas (C3R). You cannot enable cryptographic computing on existing unencrypted tables — data must be re-encrypted and re-uploaded.

## Guardrail 8: AGGREGATION Rules Require Minimum Aggregation Thresholds
AGGREGATION analysis rules enforce minimum aggregation thresholds (e.g., minimum number of rows per group). Queries that would return results below the threshold are rejected. This prevents re-identification of individual records.

## Guardrail 9: LIST Rules Control Which Columns Can Be Output
LIST analysis rules specify which columns can appear in query output and which columns can be used for joins. Attempting to SELECT columns not in the allowed list causes query failure. Review the analysis rule before writing queries.

## Guardrail 10: Custom Analysis Rules Use Approved Query Templates
CUSTOM analysis rules restrict queries to pre-approved SQL templates. Only queries matching the template structure are allowed. Ad-hoc queries are rejected. Templates must be created and approved by the table owner.

## Guardrail 11: ML Features Use Separate API and Permissions
Clean Rooms ML features (training datasets, lookalike models) use the `cleanrooms-ml` API, not the core `cleanrooms` API. They require separate IAM permissions and have their own resource types and quotas.

## Guardrail 12: Collaboration Deletion Requires All Members to Leave
A collaboration cannot be deleted while it has active members (other than the creator). All members must leave or be removed before the creator can delete the collaboration. Active queries must complete first.
