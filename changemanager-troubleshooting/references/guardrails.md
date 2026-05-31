# AWS SSM Change Manager Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Change Manager issue.

## Guardrail 1: SNS Topic Required for Notifications
Change Manager requires an SNS topic for sending approval notifications. Without a configured SNS topic, approvers won't receive email/SMS notifications about pending change requests. Verify SNS topic configuration in Change Manager settings.

## Guardrail 2: All Approvers at Each Level Must Approve
Approval workflows can have multiple levels with multiple approvers per level. ALL approvers at a given level must approve before the request advances to the next level. There is no majority-vote option. A single rejection at any level rejects the entire request.

## Guardrail 3: Templates Must Be Approved Before Use
Change templates go through their own approval process before they can be used to create change requests. A template in PENDING or REJECTED status cannot be used. Verify template status: aws ssm describe-document --name <template>.

## Guardrail 4: Scheduled Changes Require Prior Approval
Scheduled changes only execute at the specified time if all required approvals are complete. If approvals are still pending at the scheduled time, the change does NOT execute. It remains pending until approved, then executes at the next available window.

## Guardrail 5: Runbook Failures Don't Auto-Rollback
If the automation runbook fails during execution, Change Manager does NOT automatically roll back changes. Rollback logic must be built into the runbook itself. Always include error handling and rollback steps in change runbooks.

## Guardrail 6: Organization-Wide Settings Need Admin Account
Multi-account Change Manager configuration requires the Organizations management account or a delegated administrator. Member accounts inherit organization-wide settings but can have account-specific overrides.

## Guardrail 7: Change Calendar Affects Execution
If SSM Change Calendar is configured, changes are blocked during restricted periods regardless of approval status. A fully approved change will not execute during a calendar-blocked period. Check calendar state: aws ssm get-calendar-state --calendar-names <name>.

## Guardrail 8: Change Freeze Windows Block All Changes
Change freeze windows (via Change Calendar) block ALL change requests, not just specific types. Emergency changes may need to bypass the calendar, which requires specific IAM permissions and is audited.

## Guardrail 9: Approval Expiration Is Configurable
Change requests can have approval expiration times. If approvers don't respond within the configured period, the request may expire. Check template configuration for approval timeout settings.

## Guardrail 10: CloudTrail Logs All Change Manager Actions
All Change Manager operations (template creation, request submission, approvals, executions) are logged in CloudTrail. Use CloudTrail for auditing and troubleshooting: aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm.amazonaws.com.

## Guardrail 11: IAM Controls Who Can Approve
Approvers are defined in the change template. Only IAM principals listed as approvers can approve requests. IAM policies control who can create templates, submit requests, and approve changes. These are separate permissions.

## Guardrail 12: Change Requests Are OpsItems
Change requests are stored as OpsItems in OpsCenter with type /aws/changerequest. You can query them using OpsCenter APIs. This means OpsCenter must be enabled for Change Manager to function.
