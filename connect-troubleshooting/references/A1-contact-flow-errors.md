---
title: "A1 — Contact Flow Errors"
description: "Diagnose errors during contact flow execution"
status: active
severity: HIGH
triggers:
  - "Contact flow error"
  - "Flow execution failed"
  - "Contact disconnected unexpectedly"
  - "Error branch triggered"
owner: devops-agent
objective: "Identify the failing block in the contact flow and fix the error"
context: "Contact flow errors occur when a block fails during execution (Lambda timeout, invalid attribute, missing resource). The flow follows the Error branch if configured, or disconnects the contact. Contact flow logs are essential for diagnosis."
---

## Phase 1 — Triage

MUST:
- Verify contact flow logging is enabled (instance AND flow level)
- Check contact flow logs: `aws logs filter-log-events --log-group-name /aws/connect/<instance-name> --filter-pattern "ERROR"`
- Identify the failing block from the log entry (block name, type, and error)
- Get the contact flow definition: `aws connect describe-contact-flow --instance-id <id> --contact-flow-id <flow-id>`

SHOULD:
- Check if the error is consistent or intermittent
- Identify the contact ID and trace the flow execution path
- Check if the error started after a recent flow change

MAY:
- Test the flow with a test call and monitor logs in real time
- Check if the error correlates with a specific input or customer segment

## Phase 2 — Remediate

MUST:
- Fix the failing block (correct Lambda ARN, fix attribute reference, update resource)
- Ensure all blocks have Error branches configured to handle failures gracefully
- Publish the updated flow and test

SHOULD:
- Add "Set logging behavior" block at the start of the flow if not present
- Add error handling prompts ("We're experiencing difficulties, please try again")
- Test all branches of the flow, including error paths

MAY:
- Implement a fallback flow for critical errors
- Add CloudWatch alarms for contact flow error rates

## Common Issues

- symptoms: "Contact disconnects immediately after entering flow"
  diagnosis: "First block in the flow has an error and no Error branch."
  resolution: "Check the first block. Common: missing prompt, invalid Lambda ARN, or unconfigured queue."

- symptoms: "Flow works for some callers but not others"
  diagnosis: "Conditional branch fails for certain attribute values."
  resolution: "Check attribute-based branching. Ensure all possible values have matching conditions."

- symptoms: "Error after Lambda invocation"
  diagnosis: "Lambda returned an error or timed out (8-second limit)."
  resolution: "Check Lambda function logs. Optimize to complete within 8 seconds."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
3. If Lambda or Lex integration was changed, restore the original function/bot association in the contact flow
4. If agent routing or security profiles were modified, restore original profile assignments
## Output Format

```yaml
root_cause: "contact_flow_error — <block_name>: <error_detail>"
evidence:
  - type: contact_flow_logs
    content: "<error log entry>"
severity: HIGH
mitigation:
  immediate: "Fix the failing block and publish the flow"
  long_term: "Add error handling to all blocks and implement monitoring"
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
