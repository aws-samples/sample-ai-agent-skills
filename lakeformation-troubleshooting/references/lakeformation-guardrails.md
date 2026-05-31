# AWS Lake Formation Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Lake Formation issue.

## Guardrail 1: Lake Formation Permissions Work ON TOP of IAM
LF permissions are an additional layer on top of IAM. Both IAM AND LF must allow access for a principal to access data. Granting LF permissions without corresponding IAM permissions results in access denied. Similarly, IAM permissions alone are insufficient if LF permissions are enforced (IAMAllowedPrincipals removed).

## Guardrail 2: IAMAllowedPrincipals Bypasses LF Permissions by Default
New databases and tables automatically grant ALL permissions to the IAMAllowedPrincipals group. This means IAM policies alone control access. To enforce LF permissions, you MUST revoke IAMAllowedPrincipals grants. Check with: aws lakeformation list-permissions --resource '{"Database":{"Name":"<db>"}}'.

## Guardrail 3: Data Locations Must Be Registered with Lake Formation
LF can only manage fine-grained permissions on S3 locations that are registered. Unregistered locations fall back to IAM-only access. Register locations with: aws lakeformation register-resource --resource-arn <s3-arn>. The registration role must have S3 access to the location.

## Guardrail 4: Cross-Account Sharing Requires RAM
LF cross-account permission grants create RAM resource shares under the hood. The recipient account must accept the RAM invitation (unless organization sharing is enabled). Both the granting and receiving accounts need proper LF and RAM configuration. Named resources and LF-Tags can be shared cross-account.

## Guardrail 5: LF-Tags Inherit Down the Resource Hierarchy
LF-Tags assigned to a database are inherited by all tables in that database. Tags on tables are inherited by all columns. Explicit tags on child resources override inherited tags. Tag-based permissions evaluate against the effective tags (inherited + explicit). Changing a database tag affects all tables.

## Guardrail 6: Column-Level Permissions Restrict Visible Columns
When column-level permissions are granted, the principal can ONLY see the specified columns. Queries referencing non-permitted columns fail. Column permissions work with include lists (only these columns) or exclude lists (all columns except these). This affects Athena, Redshift Spectrum, and EMR queries.

## Guardrail 7: Governed Tables Use ACID Transaction Model
Governed tables support read/write transactions with snapshot isolation. Concurrent transactions can conflict, causing TransactionCanceledException. Transactions have a timeout (default varies). Compaction is needed to merge small files created by transactions. Governed tables have different storage costs than standard tables.

## Guardrail 8: LF Administrators Have Unrestricted Permissions
Lake Formation administrators can grant/revoke any permission, register any location, and manage all catalog resources. At least one LF admin must be configured. The IAM administrator is NOT automatically an LF admin. LF admins are configured in data lake settings. Multiple admins can be designated.

## Guardrail 9: Blueprint Execution Requires a Specific IAM Role
Blueprints create Glue workflows that ingest data into the data lake. The blueprint execution role needs permissions for Glue, S3, LF, and the data source. Blueprint failures often stem from insufficient role permissions. The role must be trusted by both Glue and Lake Formation services.

## Guardrail 10: Data Filters Evaluate at Query Time
Data cell filters (row and column filters) are evaluated when queries are executed, not when permissions are granted. Filter expressions use PartiQL syntax. Complex filters can impact query performance. Filters are applied per-principal and can be combined with table-level permissions.

## Guardrail 11: Hybrid Permission Mode Allows Gradual Migration
Lake Formation supports a hybrid mode where some databases/tables use LF permissions and others use IAM-only. This is controlled by the IAMAllowedPrincipals grants per resource. You can migrate to LF permissions incrementally by removing IAMAllowedPrincipals from individual resources.

## Guardrail 12: Catalog Resource Hierarchy Affects Permission Inheritance
The Glue Data Catalog has a hierarchy: Catalog → Database → Table → Column. Permissions granted at a higher level can include grant options for lower levels. Database-level CREATE_TABLE permission allows creating tables. Table-level permissions can be granted with or without column restrictions.
