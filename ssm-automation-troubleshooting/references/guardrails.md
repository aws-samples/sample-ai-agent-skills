# SSM Automation Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any SSM Automation issue.

## Guardrail 1: Simple Execution vs Rate Control Are Different Modes
Simple execution runs steps sequentially on a single target. Rate Control execution runs against multiple targets with MaxConcurrency and MaxErrors parameters. Failure behavior, logging, and troubleshooting differ significantly between modes. Do not conflate them.

## Guardrail 2: $DEFAULT vs $LATEST Document Versions
Automation executions use the $DEFAULT document version unless explicitly specified. Updating a document creates a new version but does NOT change $DEFAULT. You must explicitly set the default version with update-document-default-version. Never assume the latest version is what executions use.

## Guardrail 3: Approval Steps Have Configurable Timeouts
Approval steps wait for manual approval with a configurable timeout (default 7 days). If no approver acts within the timeout, the execution times out and fails. Approvals do NOT auto-approve or auto-reject. The timeout is set in the document, not at execution time.

## Guardrail 4: Cross-Account Requires Organizations and IAM Roles
Cross-account automation requires AWS Organizations with a management account or delegated admin. Target accounts must have IAM roles with trust policies allowing the management account to assume them. Standalone accounts cannot use cross-account automation.

## Guardrail 5: Change Calendar Blocks Entire Executions
When a Change Calendar is CLOSED, automation executions that reference it are blocked entirely with CalendarStateIsClosedException. The calendar does not modify or delay executions — it blocks them. Check calendar state before troubleshooting permission errors.

## Guardrail 6: Output Parameters Use Step Reference Syntax
Output parameters from steps are referenced as {{stepName.outputKey}} in subsequent steps. Incorrect references produce empty values, not errors. Always verify the exact step name and output key name. Output keys are case-sensitive.

## Guardrail 7: Service Roles Need Both SSM and Target Permissions
The automation service role (AssumeRole parameter) needs ssm:* permissions AND permissions for every AWS action the document performs. A role with only SSM permissions will fail on steps that call EC2, S3, Lambda, or other services.

## Guardrail 8: Multi-Region Execution Uses Separate Executions Per Region
Multi-region automation creates separate child executions in each target region. Each regional execution can succeed or fail independently. A failure in one region does not automatically stop executions in other regions unless configured.

## Guardrail 9: Maintenance Window Tasks Have Their Own Service Role
Maintenance window automation tasks use the maintenance window service role, not the automation document's AssumeRole. Both roles must have appropriate permissions. Confusion between these roles is a common source of permission errors.

## Guardrail 10: Automation Execution Has a 30-Day Timeout
Automation executions have a maximum timeout of 30 days (25 hours for simple executions without long-running steps). Long-running automations that exceed this timeout are automatically cancelled. Plan multi-step workflows accordingly.

## Guardrail 11: Step Types Have Different Failure Behaviors
Different step types (aws:executeScript, aws:runCommand, aws:invokeLambdaFunction, aws:approve, etc.) have different failure behaviors, timeout defaults, and retry mechanisms. Do not assume all steps fail the same way.

## Guardrail 12: Automation Documents Are Regional Resources
SSM Automation documents are regional. A document created in us-east-1 is not available in eu-west-1. For multi-region automation, documents must exist in each target region. Use CloudFormation StackSets or manual replication.
