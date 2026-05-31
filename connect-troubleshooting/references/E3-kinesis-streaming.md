---
title: "E3 — Kinesis Data Streaming Issues"
description: "Diagnose issues with CTR and agent event streaming to Kinesis"
status: active
severity: MEDIUM
triggers:
  - "CTR streaming not working"
  - "Kinesis stream empty"
  - "Agent events not streaming"
  - "Data streaming error"
owner: devops-agent
objective: "Restore data streaming from Connect to Kinesis"
context: "Connect can stream CTRs and agent events to Kinesis Data Streams or Kinesis Data Firehose. This enables near-real-time analytics. Failures include: stream not configured, permission errors, stream throttling, or data format issues."
---

## Phase 1 — Triage

MUST:
- Check if data streaming is enabled on the instance: `aws connect list-instance-attributes --instance-id <id>`
- Verify the Kinesis stream/Firehose exists and is active
- Check the Connect instance's data streaming configuration
- Verify IAM permissions for Connect to write to Kinesis

SHOULD:
- Check Kinesis stream metrics (IncomingRecords, WriteProvisionedThroughputExceeded)
- Verify the stream has sufficient shards for the data volume
- Check if the stream was recently modified or recreated

MAY:
- Check CloudTrail for streaming configuration changes
- Test by making a call and checking for the CTR in the stream

## Phase 2 — Remediate

MUST:
- Enable data streaming on the instance if not configured
- Fix IAM permissions for Connect to write to the stream
- Verify the stream ARN in the configuration is correct

SHOULD:
- Increase stream capacity if throttling
- Verify the consumer application is reading from the stream
- Test end-to-end data flow after fix

MAY:
- Implement stream monitoring and alerting
- Set up dead letter handling for failed records

## Common Issues

- symptoms: "No CTRs appearing in Kinesis stream"
  diagnosis: "Data streaming not enabled or stream ARN incorrect."
  resolution: "Enable CTR streaming in instance settings with the correct stream ARN."

- symptoms: "Some CTRs missing from stream"
  diagnosis: "Kinesis stream throttling due to insufficient capacity."
  resolution: "Increase the number of shards or switch to on-demand capacity mode."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
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
2. If agent routing or security profiles were modified, restore original profile assignments
3. If recording or streaming configuration was changed, restore original storage settings
4. If authentication or permission settings were changed, revert identity provider configuration
## Output Format

```yaml
root_cause: "kinesis_streaming — <specific cause>"
evidence:
  - type: instance_config
    content: "<streaming configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix streaming configuration"
  long_term: "Implement stream monitoring and capacity planning"
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
