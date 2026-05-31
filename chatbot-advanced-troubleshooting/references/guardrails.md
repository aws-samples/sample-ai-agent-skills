# AWS Chatbot Advanced Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any advanced AWS Chatbot issue.

## Guardrail 1: Slack and Teams Have Different Integration Models
Slack uses OAuth 2.0 with Chatbot-managed app installation. Microsoft Teams uses Azure AD app registration with tenant-level consent. Setup flows, permissions, and troubleshooting differ significantly. Do not apply Slack troubleshooting to Teams issues or vice versa.

## Guardrail 2: Channel Guardrails Are Not IAM Policies
Channel guardrails use IAM policy syntax but are evaluated by Chatbot, not IAM. They restrict what commands and actions can be run from a specific chat channel. Modifying IAM policies does not change guardrail behavior. Guardrails are configured per channel configuration.

## Guardrail 3: Custom Actions Require Both IAM and Configuration
Custom actions need two things: the action must be defined in the channel configuration AND the channel IAM role must have permissions for the action's underlying AWS API calls. Missing either one causes the action to fail silently or with a generic error.

## Guardrail 4: Run Command From Chat Uses the Channel Role
When running SSM commands from chat, Chatbot uses the channel's IAM role to call ssm:SendCommand. The role needs SSM permissions AND the target instances must have SSM agent running. This is different from running commands with a separate SSM service role.

## Guardrail 5: Cross-Account Notifications Require SNS Topic Policy
For cross-account notifications, the SNS topic in the source account must have a resource policy allowing the Chatbot account to subscribe. Simply adding the topic ARN to the Chatbot configuration is not sufficient without the topic policy.

## Guardrail 6: Event Pattern Matching Failures Are Silent
If an EventBridge event pattern does not match incoming events, notifications are silently dropped. There is no error or log entry for non-matching events. Always test event patterns with sample events before deploying.

## Guardrail 7: Lambda Invocation From Chat Has Size Limits
Lambda functions invoked from chat have response size limits. Large responses are truncated in the chat display. The Lambda function must return a response formatted for chat display (markdown for Slack, adaptive cards for Teams).

## Guardrail 8: Audit Logging Captures Actions Not Messages
CloudTrail logs Chatbot API calls and command executions, but does NOT log the actual chat message content. You can see what commands were run and by whom, but not the conversational context around them.

## Guardrail 9: Teams Workspace Must Be Configured Before Channels
Microsoft Teams integration requires a workspace (tenant) configuration before individual channel configurations can be created. The workspace maps to an Azure AD tenant. Channel configurations reference the workspace.

## Guardrail 10: Notification Formatting Differs by Platform
Notifications are formatted differently for Slack and Teams. Slack uses Block Kit, Teams uses Adaptive Cards. Custom notification formatting must account for the target platform. A format that works in Slack may not render correctly in Teams.

## Guardrail 11: Chatbot Operates in Specific Regions Only
AWS Chatbot configurations are created in specific regions. Not all regions support Chatbot. The Chatbot service endpoint may differ from the region where monitored resources exist. Verify regional availability before troubleshooting.

## Guardrail 12: Action Approvals Require Specific IAM Permissions
Action approvals in Chatbot require the approver to have specific IAM permissions. The approval flow is different from SSM Automation approvals. Chatbot approvals are channel-specific and use the channel's guardrail configuration.
