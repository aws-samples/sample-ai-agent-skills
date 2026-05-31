---
title: "Z1 — General Amazon Connect Troubleshooting (Catch-All)"
description: "Fallback SOP for Connect issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Connect issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check instance status: `aws connect describe-instance --instance-id <id>`
- Check instance attributes (enabled features): `aws connect list-instance-attributes --instance-id <id>`
- Check contact flow logs for errors (if enabled)
- Check real-time metrics: agents available, contacts in queue
- Check CloudWatch metrics for anomalies (CallsPerInterval, MisconfiguredPhoneNumbers)
- Verify phone numbers are claimed and associated with flows

SHOULD:
- Check recent contact details for disconnect reasons
- Verify agent states and CCP connectivity
- Check queue configurations and hours of operation
- Review recent configuration changes via CloudTrail

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Contact flow errors → Contact Flow (A1-A4)
- Routing issues → Call Routing (B1-B4)
- Agent problems → Agent Issues (C1-C4)
- Telephony issues → Telephony (D1-D4)
- Integration failures → Integrations (E1-E4)
- Channel issues → Channels (F1-F3)
- Metrics gaps → Metrics & Reporting (G1-G3)
- Access problems → Security & Access (H1-H3)
- Number issues → Phone Numbers (I1-I3)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all contact flows for errors
- Review all queue configurations
- Check all phone number associations
- Verify all integrations (Lambda, Lex, Kinesis)
- Check agent routing profiles and security profiles

SHOULD:
- Check AWS Health Dashboard for Connect service events
- Review CloudTrail for recent configuration changes
- Compare with a known-good configuration

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
```

## Escalation Conditions

- Contact center is live with active calls or chats in progress
- Issue affects agent ability to receive or handle contacts
- Fix requires modifying a contact flow associated with a phone number
- Resolution involves changes to hours of operation during business hours
- Issue impacts call recording, CTR generation, or compliance requirements

## Data Sensitivity

HIGH - Contact Trace Records (CTRs) contain customer phone numbers, call timestamps, and interaction details. Call recordings contain customer voice data. Chat transcripts may contain PII, financial data, or healthcare information. Agent data includes employee performance metrics.

## Prohibited Actions

- NEVER suggest deleting a contact flow that is associated with a phone number - this breaks inbound routing
- NEVER suggest modifying production hours of operation during active business hours
- NEVER recommend removing security profiles from agents while they are logged in
- NEVER suggest disabling call recording on queues that have compliance requirements
- NEVER recommend deleting or modifying a published contact flow without first testing in a non-production flow

## Phase 3 - Rollback

1. If a contact flow was modified, publish the previous saved version of the flow from the Connect console
2. If queue configuration was changed, restore original routing profile assignments and queue settings
3. If hours of operation were modified, revert to the original schedule immediately
4. If Lambda or Lex integration was changed, restore the original function/bot association in the contact flow
5. If agent routing or security profiles were modified, restore original profile assignments
6. If phone number associations were changed, reassociate the number with the original contact flow
7. If recording or streaming configuration was changed, restore original storage settings
8. If authentication or permission settings were changed, revert identity provider configuration
## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<contact_flow|routing|agent|telephony|integration|channel|metrics|access|numbers|unknown>"
investigation_path: "instance status → metrics → flows → <domain_classification>"
evidence:
  - type: instance_status
    content: "<instance details>"
  - type: metrics
    content: "<real-time metric values>"
  - type: contact_flow_logs
    content: "<error entries>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
