# Health Dashboard Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Health Dashboard issue.

## Guardrail 1: Health API Requires Business or Enterprise Support
The AWS Health API requires Business or Enterprise Support plan. Basic and Developer plans can only view health events through the console. API calls from unsupported plans return SubscriptionRequiredException.

## Guardrail 2: API Endpoint Is us-east-1 Only
The Health API is only available in us-east-1 (and us-gov-west-1 for GovCloud). All API calls must specify --region us-east-1. This applies to both single-account and organizational health API operations.

## Guardrail 3: Three Event Categories Have Different Meanings
Health events fall into three categories: issue (active service disruption), accountNotification (account-specific information), and scheduledChange (planned maintenance). Each category has different urgency and response requirements. Never conflate categories.

## Guardrail 4: Public Events Do Not List Specific Resources
Public health events (service-wide issues) affect all customers in a region and do not list specific affected entities. Only account-specific events (accountNotification) list specific affected resources. Never claim you can identify specific resources for public events.

## Guardrail 5: Events Retained for 90 Days Only
Health events are accessible via API for 90 days from the event's last update. Events older than 90 days are not queryable. For long-term event history, events must be archived to S3 or a database via EventBridge integration.

## Guardrail 6: Affected Entities May Not Be Immediately Available
When a health event is first published, the list of affected entities may be empty or incomplete. AWS populates affected entities as it identifies impacted resources. Do not conclude no resources are affected based on an empty initial entity list.

## Guardrail 7: EventBridge Health Events Are Region-Specific
Health events are published to EventBridge in us-east-1 for global events and in the affected region for regional events. An EventBridge rule in us-east-1 catches global events but may miss regional events. Rules must be created in relevant regions.

## Guardrail 8: Organizational Health Requires All Features Enabled
Organizational health view requires AWS Organizations with all features enabled and Health enabled as a trusted service. Consolidated billing only is not sufficient. The management account or delegated administrator can access organizational health.

## Guardrail 9: Scheduled Changes May Not Have Fixed End Times
Scheduled change events include a start time but may not include a specific end time. Some maintenance windows are open-ended. Never assume a scheduled change has a fixed duration unless the event explicitly states one.

## Guardrail 10: Health Events Are Not Real-Time Alerts
Health events are published as AWS identifies issues. There can be a delay between the start of an issue and the publication of the health event. Health Dashboard is not a real-time monitoring tool. Use CloudWatch for real-time alerting.

## Guardrail 11: Account-Specific Events Require Account Context
Account-specific health events are only visible to the affected account. Other accounts in the same organization cannot see account-specific events unless organizational health view is enabled. Never assume cross-account event visibility without org setup.

## Guardrail 12: Event Status Transitions Are One-Way
Health events transition from upcoming → open → closed. Events do not revert to previous states. A closed event means the issue is resolved or the maintenance is complete. New occurrences create new events.
