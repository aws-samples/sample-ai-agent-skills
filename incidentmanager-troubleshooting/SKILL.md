---
name: incidentmanager-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Systems Manager
  Incident Manager problems by analyzing response plans, incident
  workflows, contacts, and following structured runbooks. Activate when:
  Response plan creation failures, incident creation issues, escalation
  plan problems, contact channel failures, engagement plan errors,
  automation runbook failures, manual step issues, timeline event
  problems, metric integration issues, CloudWatch alarm integration
  failures, EventBridge rule problems, post-incident analysis errors,
  action item tracking issues, or the user says something is wrong
  with Incident Manager without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with SSM Incident Manager, SSM,
  CloudWatch, EventBridge, SNS, IAM, and CloudTrail permissions.
---

# Incident Manager Diagnostics

## When to use

Any Incident Manager investigation where the console alone is insufficient — response plan configuration, incident lifecycle management, contact and escalation issues, automation runbook failures, timeline and metrics problems, alarm/event integration, or post-incident analysis.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ssm-incidents list-response-plans
aws ssm-incidents list-incident-records
aws ssm-contacts list-contacts
aws ssm-contacts list-engagements
aws ssm-incidents get-response-plan --arn <plan-arn>
```

### Step 2 — Deep dive

```
aws ssm-incidents get-incident-record --arn <incident-arn>
aws ssm-incidents list-timeline-events --incident-record-arn <incident-arn>
aws ssm-incidents list-related-items --incident-record-arn <incident-arn>
aws ssm-contacts get-contact --contact-id <contact-arn>
aws ssm-contacts get-contact-channel --contact-channel-id <channel-arn>
aws ssm-incidents get-replication-set
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm-incidents.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any Incident Manager issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ssm-incidents list-response-plans` | List all response plans |
| `ssm-incidents get-response-plan` | Response plan details |
| `ssm-incidents list-incident-records` | List incidents |
| `ssm-incidents get-incident-record` | Incident details and status |
| `ssm-incidents list-timeline-events` | Incident timeline |
| `ssm-incidents list-related-items` | Related resources and items |
| `ssm-contacts list-contacts` | List contacts and escalation plans |
| `ssm-contacts get-contact` | Contact details |
| `ssm-contacts get-contact-channel` | Contact channel configuration |
| `ssm-contacts list-engagements` | Active engagements |
| `ssm-incidents get-replication-set` | Replication set configuration |

## Gotchas: Incident Manager

- Replication set must be created first: Before using Incident Manager, you must create a replication set that defines which regions are active. Without a replication set, no other operations work.
- Contact channels require activation: After creating a contact channel (email, SMS, voice), it must be activated by the recipient. Unactivated channels cannot receive notifications.
- Escalation plans are time-based: Escalation plans engage contacts in stages based on time. If the incident is acknowledged before the next stage, escalation stops. Misconfigured timing can skip contacts.
- Automation runbooks run as SSM documents: Incident Manager automation uses SSM Automation documents. The execution role must have permissions for all actions in the runbook. Missing permissions cause silent failures.
- CloudWatch alarms trigger incidents via response plans: Alarms must be configured to trigger the correct response plan. The alarm action must use the response plan ARN, not an SNS topic.
- Cross-region replication is automatic: Incident data replicates across regions in the replication set. However, response plans and contacts are regional resources that must be created in each region.
- Incident impact levels affect routing: Impact levels (1-5) determine which contacts and escalation plans are engaged. Misconfigured impact levels can route incidents to wrong teams.
- Post-incident analysis has a 24-hour delay: Analysis features become available after the incident is resolved. Some metrics and timeline data may take up to 24 hours to finalize.

## Anti-hallucination rules

1. Always cite specific response plan ARNs, incident ARNs, or API responses as evidence.
2. Replication set is required first — never suggest creating response plans without it.
3. Contact channels must be activated — never assume channels work without activation.
4. Automation runbooks need SSM permissions — never claim they run without an execution role.
5. Response plans are regional — never claim they replicate automatically.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 20 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Response | A1-A3 | Response plan, incident creation, escalation |
| B — Contacts | B1-B2 | Contact channels, engagement plans |
| C — Runbooks | C1-C2 | Automation runbooks, manual steps |
| D — Timeline | D1-D2 | Timeline events, metrics |
| E — Integration | E1-E2 | CloudWatch alarms, EventBridge |
| F — Post-Incident | F1-F2 | Analysis, action items |
| Z — Catch-All | Z1 | General Incident Manager troubleshooting |
