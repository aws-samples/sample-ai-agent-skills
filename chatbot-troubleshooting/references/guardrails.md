# AWS Chatbot Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Chatbot issue.

## Guardrail 1: Workspace/Team Authorization Is a Manual Console Step
Slack workspace authorization and Microsoft Teams tenant consent must be completed in the AWS Console. This is a one-time manual step that cannot be fully automated via CLI or API. The workspace/team must be authorized before any channel configurations can be created.

## Guardrail 2: Channel Configurations Are Per-Channel, Per-Account
Each Slack channel or Teams channel needs its own Chatbot configuration with an IAM role and SNS topic subscriptions. A single channel can have only one configuration per AWS account. Multiple AWS accounts can send to the same channel with separate configurations.

## Guardrail 3: Channel IAM Role and Guardrail Policies Are Separate Permission Layers
The channel IAM role determines what AWS actions can be performed via CLI commands in chat. Guardrail policies act as an additional permission boundary that restricts actions even further. Both must allow an action for it to succeed. Do not conflate these two permission mechanisms.

## Guardrail 4: SNS Topics Must Be Subscribed to the Chatbot Channel
Notifications flow through SNS. The Chatbot channel configuration must include the SNS topic ARN, and the topic must have a subscription to the Chatbot endpoint. Missing or unconfirmed subscriptions mean no notifications are delivered.

## Guardrail 5: Only Specific AWS Services Send Notifications Through Chatbot
Chatbot supports notifications from: CloudWatch Alarms, AWS Health events, Security Hub findings, AWS Budgets alerts, GuardDuty findings, and custom EventBridge events. Not every AWS service integrates directly with Chatbot. Do not suggest Chatbot can deliver notifications from unsupported services.

## Guardrail 6: CLI Commands in Chat Have Specific Syntax and Restrictions
Chat commands must start with `@aws` followed by the CLI command. Not all CLI commands are supported in chat. Read-only commands (describe, list, get) are allowed by default. Mutating commands (create, delete, update) require explicit guardrail policy approval. Some commands are never allowed in chat.

## Guardrail 7: Slack App Must Be Installed and Authorized
The AWS Chatbot Slack app must be installed in the Slack workspace and authorized. Removing the app, restricting its permissions, or revoking OAuth tokens breaks all channel configurations. Workspace admins control app installation.

## Guardrail 8: Teams Integration Requires Azure AD Tenant Admin Consent
Microsoft Teams integration requires an Azure AD tenant administrator to grant consent for the AWS Chatbot app. Without tenant-level consent, Teams channel configurations cannot be created. This is a one-time admin action.

## Guardrail 9: Notification Formatting Is Chatbot-Managed
You cannot customize the notification card format in Slack or Teams. Chatbot formats notifications based on the source service (CloudWatch, Security Hub, etc.). The format includes relevant details but cannot be modified with custom templates.

## Guardrail 10: Rate Limits Apply to Notification Delivery
Chatbot has rate limits on notification delivery to prevent flooding channels. High-volume alarm state changes or frequent Security Hub findings may be throttled or batched. Do not interpret throttled notifications as delivery failures.

## Guardrail 11: EventBridge Integration Requires Specific Event Patterns
Not all EventBridge events are processed by Chatbot. Chatbot processes events matching specific patterns from supported services. Custom EventBridge rules must target the Chatbot SNS topic with the correct event pattern for notifications to appear.

## Guardrail 12: Read-Only Commands Are Default — Mutating Commands Need Explicit Approval
By default, Chatbot only allows read-only CLI commands (describe, list, get). Mutating commands (create, update, delete, start, stop) require the guardrail policy to explicitly allow those actions. The default guardrail policy is ReadOnlyAccess.
