# AWS Budgets Actions Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Budgets Actions issue.

## Guardrail 1: Cost Data Has Up to 24-Hour Delay
AWS cost and usage data can be delayed up to 24 hours. Budget thresholds and actions are based on this delayed data. Do not expect real-time cost tracking or immediate action triggering.

## Guardrail 2: Actions Are Not Automatically Reversed
When a budget action executes (stops instances, applies policies), it is NOT automatically reversed when spending drops below the threshold. Stopped instances stay stopped. Applied policies stay applied. Manual intervention is required to reverse actions.

## Guardrail 3: Three Action Types with Different Requirements
Budgets Actions supports three action types: IAM policy attachment, SCP attachment, and EC2/RDS instance stop. Each requires different IAM permissions and has different impact scope.

## Guardrail 4: Actions Require a Dedicated IAM Role
Budget actions execute using a dedicated IAM role with trust policy for budgets.amazonaws.com. The role must have specific permissions for the action type (iam:AttachUserPolicy, organizations:AttachPolicy, ec2:StopInstances, etc.).

## Guardrail 5: SCP Actions Require Organizations
SCP-based budget actions can only be used in accounts that are part of AWS Organizations. SCPs must be applied from the management account or delegated admin.

## Guardrail 6: Approval Workflows Use SNS
When actions require approval, notifications are sent via SNS. The approver must explicitly approve or deny the action. Without approval, the action does not execute.

## Guardrail 7: EC2/RDS Stop Actions Are Disruptive
Stopping EC2 or RDS instances is a disruptive action that affects running workloads. Instance data on ephemeral storage is lost. Applications must be designed to handle instance stops gracefully.

## Guardrail 8: Budget Filters Affect Action Scope
Budget filters (service, linked account, tag) determine which costs are tracked. Actions trigger based on filtered costs, not total account spend. Ensure filters match the intended scope.

## Guardrail 9: Organization Budgets Require Management Account
Creating budgets for member accounts requires management account permissions or delegated admin. Member accounts cannot create budgets for other accounts.

## Guardrail 10: Multiple Actions Can Trigger Simultaneously
A single budget can have multiple actions at different thresholds. All actions whose thresholds are exceeded will trigger. Plan action ordering carefully.

## Guardrail 11: Budget Period Affects Threshold Calculation
Budget periods (monthly, quarterly, annually) affect how thresholds are calculated. A 80% threshold on a monthly budget triggers at different absolute amounts than on an annual budget.

## Guardrail 12: Notification Subscribers Must Confirm
SNS email subscribers must confirm their subscription before receiving notifications. Unconfirmed subscribers will not receive budget alerts or approval requests.
