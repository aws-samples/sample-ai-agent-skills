# Amazon AppFlow Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AppFlow issue.

## Guardrail 1: Flow Status vs Execution Status Are Different
A flow can be ACTIVE (enabled) but have failed executions. Flow status indicates whether the flow is configured and enabled. Execution status indicates whether individual runs succeeded. Do not confuse flow-level status with execution-level status.

## Guardrail 2: Connector Profiles Are Region-Specific
Connector profiles are created per region. A Salesforce connector profile in us-east-1 cannot be used by a flow in eu-west-1. Each region requires its own connector profile, even for the same SaaS source.

## Guardrail 3: OAuth Token Refresh Is Not Automatic for All Connectors
Some connectors (e.g., Salesforce) use OAuth refresh tokens that can expire if the Connected App configuration changes, the user revokes access, or the refresh token lifetime is exceeded. Do not assume tokens refresh indefinitely.

## Guardrail 4: S3 Destination Uses Structured Prefixes
AppFlow writes to S3 using a structured prefix pattern: `<configured-prefix>/<flow-name>/<execution-id>/`. Files are not written to a flat path. Do not assume the output location without checking the actual prefix structure.

## Guardrail 5: Redshift Requires Intermediate S3 Staging
Redshift destinations require an intermediate S3 bucket for data staging. AppFlow writes data to S3 first, then uses COPY to load into Redshift. Both S3 bucket permissions and Redshift COPY IAM role must be correctly configured.

## Guardrail 6: Scheduled Flows Use UTC and EventBridge Syntax
All scheduled flow times are in UTC. Cron expressions follow AWS EventBridge syntax (6 fields), not standard Unix cron (5 fields). The minimum schedule interval is 1 minute for some connectors but varies by source type.

## Guardrail 7: Data Mapping Errors May Be Source Schema Changes
Data mapping failures often result from source schema changes (e.g., a Salesforce field was renamed or removed). AppFlow does not automatically detect source schema changes. Re-fetch the source schema to update mappings.

## Guardrail 8: Custom Connectors Must Follow SDK Response Format
Custom connectors built with Lambda must return responses in the exact AppFlow Custom Connector SDK format. Missing fields, incorrect types, or malformed pagination tokens cause execution failures that may not have clear error messages.

## Guardrail 9: Private Connections Require VPC Endpoint Health
Private connections use AWS PrivateLink VPC endpoints. The endpoint must be in AVAILABLE state, in the same region as the flow, and the security group must allow traffic on the required ports. DNS resolution must also work correctly.

## Guardrail 10: KMS Encryption Applies to Data at Rest Only
AppFlow encrypts data at rest using KMS. Data in transit uses TLS. The KMS key must grant appflow.amazonaws.com permission to GenerateDataKey and Decrypt. Cross-account KMS keys require additional key policy configuration.

## Guardrail 11: Flow Execution Limits Vary by Connector
Different connectors have different API rate limits, record size limits, and batch size limits. Salesforce has API call limits per 24-hour period. Exceeding source API limits causes throttling errors, not AppFlow errors.

## Guardrail 12: AppFlow Does Not Transform Data Types Automatically
AppFlow maps fields between source and destination but does not automatically convert incompatible data types. A Salesforce picklist field mapped to a Redshift INTEGER column will fail. Verify type compatibility before execution.
