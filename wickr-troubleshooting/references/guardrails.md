# AWS Wickr Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Wickr issue.

## Guardrail 1: End-to-End Encryption Means AWS Cannot Read Messages
AWS Wickr uses end-to-end encryption. AWS does not have access to message content, even for troubleshooting. Do not suggest that AWS Support can retrieve or read message content. Only message metadata (delivery status, timestamps) is available for diagnostics.

## Guardrail 2: Wickr Networks Are Not VPCs
A Wickr network is a logical grouping of users, settings, and policies. It is NOT an AWS VPC or networking construct. Do not confuse Wickr network configuration with VPC subnets, security groups, or route tables.

## Guardrail 3: Message Retention Is Opt-In
By default, Wickr messages are ephemeral and not retained after delivery. Message retention must be explicitly configured with an S3 bucket and KMS key. Do not assume messages are stored or recoverable unless retention is enabled.

## Guardrail 4: Admin Console Requires SSO or SAML
The Wickr admin console requires IAM Identity Center (SSO) or SAML-based federation for authentication. Direct IAM user credentials cannot be used to log into the admin console. Do not suggest IAM user login for admin access.

## Guardrail 5: Bots Are External Processes Not Lambda Functions
Wickr bots run as external processes using the Wickr Bot SDK. They are NOT AWS Lambda functions or Step Functions. Bots connect to the Wickr network via API and must be hosted on compute infrastructure (EC2, ECS, etc.).

## Guardrail 6: Federation Requires Mutual Configuration
Federation between Wickr networks requires both network administrators to configure and approve the federation. One-sided configuration does not enable cross-network messaging. Both sides must complete setup.

## Guardrail 7: File Sharing Limits Are Network-Level Settings
File sharing size limits and policies are configured at the Wickr network level by administrators. Individual users cannot override these limits. Do not suggest user-level workarounds for network-level restrictions.

## Guardrail 8: Compliance Logging Captures Metadata Not Content
Compliance logging captures message metadata (sender, recipient, timestamp, delivery status) but does NOT capture message content due to end-to-end encryption. Do not suggest compliance logs contain message text.

## Guardrail 9: User Provisioning Supports Multiple Methods
Users can be provisioned via the admin console, SCIM (System for Cross-domain Identity Management), or directory sync. Each method has different requirements and limitations. Do not assume all provisioning methods are equivalent.

## Guardrail 10: KMS Key Is Required for Retention
Message retention requires a KMS key for encrypting retained data in S3. The KMS key must be in the same region as the Wickr network. Do not configure retention without verifying KMS key availability and permissions.

## Guardrail 11: Network Deletion Is Irreversible
Deleting a Wickr network permanently removes all network configuration, user associations, and settings. Retained messages in S3 are preserved but the network cannot be restored. Do not suggest network deletion as a troubleshooting step.

## Guardrail 12: API Access Uses Wickr-Specific Endpoints
Wickr API access uses Wickr-specific endpoints and authentication, not standard AWS API Gateway endpoints. Bot and integration API calls follow the Wickr Bot SDK patterns, not AWS SDK patterns.
