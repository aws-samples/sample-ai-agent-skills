# Incident Manager Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Incident Manager issue.

## Guardrail 1: Replication Set Must Be Created First
Before any Incident Manager operations (response plans, contacts, incidents), a replication set must be created. The replication set defines active regions and encryption. Without it, all API calls fail. Check with `get-replication-set`.

## Guardrail 2: Contact Channels Require Activation
After creating a contact channel (email, SMS, voice), the recipient must activate it by responding to a verification message. Unactivated channels silently fail to deliver notifications during incidents. Always verify activation status.

## Guardrail 3: Escalation Plans Are Time-Based with Acknowledgment Gates
Escalation plans engage contacts in stages based on elapsed time. If the incident is acknowledged before the next stage timer expires, escalation stops. Misconfigured stage durations can cause contacts to be skipped or engaged too late.

## Guardrail 4: Automation Runbooks Require SSM Execution Role
Incident Manager automation uses SSM Automation documents. The execution role specified in the response plan must have permissions for all actions in the runbook. Missing permissions cause the runbook to fail silently or partially.

## Guardrail 5: Response Plans Are Regional Resources
Response plans, contacts, and escalation plans are regional resources. They do not automatically replicate across regions in the replication set. You must create them in each region where incidents may occur.

## Guardrail 6: CloudWatch Alarm Actions Must Reference Response Plan ARN
To trigger an incident from a CloudWatch alarm, the alarm action must be the response plan ARN (not an SNS topic). The alarm must have the correct IAM permissions to invoke the response plan.

## Guardrail 7: Impact Levels Determine Routing and Priority
Incident impact levels (1=Critical to 5=No Impact) determine which contacts and escalation plans are engaged. Response plans can have different engagement configurations per impact level. Wrong impact level routes incidents to wrong teams.

## Guardrail 8: Cross-Region Incident Data Replicates Automatically
Incident records, timeline events, and related items replicate across regions in the replication set. However, this replication is eventual — there may be brief delays. Do not create duplicate incidents in multiple regions.

## Guardrail 9: Post-Incident Analysis Requires Resolved Status
Post-incident analysis features (metrics, questions, action items) are only available after the incident status is set to RESOLVED. Analysis data may take up to 24 hours to finalize after resolution.

## Guardrail 10: SNS Topics for Contact Channels Must Be in Same Region
SNS topics used for contact channel notifications must be in the same region as the contact. Cross-region SNS topics are not supported for contact channel delivery.

## Guardrail 11: SSM Automation Documents Must Exist Before Response Plan Reference
Response plans reference SSM Automation documents by name or ARN. The document must exist before the response plan is created or updated. Referencing a non-existent document causes response plan creation to fail.

## Guardrail 12: Engagement Plans Cannot Be Modified During Active Engagement
Once an engagement is active (contacts are being notified), the engagement plan cannot be modified for that engagement. Changes to the plan only affect future engagements. To change routing for an active incident, manually engage contacts.
