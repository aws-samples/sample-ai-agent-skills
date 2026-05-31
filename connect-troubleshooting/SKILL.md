---
name: connect-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Connect
  problems by analyzing contact flows, agent states, telephony
  issues, and following structured runbooks. Activate when: calls
  are not routing correctly, agents cannot log in or receive calls,
  contact flows have errors, DTMF or IVR is not working, call
  quality is poor (latency, jitter, packet loss), outbound calls
  fail, queues are overflowing, CTR data is missing, Lex bots are
  not responding, Lambda integrations fail within contact flows,
  real-time or historical metrics are incorrect, chat or task
  channels have issues, SAML/SSO login fails, CCP connectivity
  problems, phone number porting or claiming issues, or the user
  says something is wrong with Connect without naming specific
  symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Amazon Connect, CloudWatch,
  CloudWatch Logs, Lambda, Lex, Kinesis, S3, and IAM permissions.
  For CCP troubleshooting, browser-level diagnostics may be needed.
---

# Amazon Connect Diagnostics

## When to use

Any Amazon Connect investigation where the console alone is insufficient — call routing failures, agent state issues, contact flow errors, telephony quality problems, integration failures (Lambda, Lex), CCP connectivity, metrics gaps, or channel-specific issues (voice, chat, tasks).

## Investigation workflow

### Step 1 — Collect and triage

```
# Get instance details
aws connect describe-instance --instance-id <instance-id>

# Check instance status and configuration
aws connect list-instance-attributes --instance-id <instance-id>

# Get recent contact details (if contact ID known)
aws connect describe-contact --instance-id <instance-id> --contact-id <contact-id>

# Check current agent status
aws connect get-current-metric-data --instance-id <instance-id> \
  --filters '{"Queues":["<queue-arn>"]}' \
  --current-metrics '[{"Name":"AGENTS_AVAILABLE","Unit":"COUNT"},{"Name":"AGENTS_ON_CALL","Unit":"COUNT"},{"Name":"CONTACTS_IN_QUEUE","Unit":"COUNT"}]'

# Check contact flow logs (if enabled)
aws logs filter-log-events --log-group-name /aws/connect/<instance-name> --filter-pattern "ERROR"
```

Triage returns:
- Instance configuration and enabled features
- Contact details and disconnect reasons
- Agent availability and queue state
- Contact flow execution errors

If the instance itself is unreachable, that IS the root cause domain. Don't chase routing or agent symptoms.

### Step 2 — Domain deep dive (only if needed)

```
# Contact flow details
aws connect describe-contact-flow --instance-id <instance-id> --contact-flow-id <flow-id>

# Queue configuration
aws connect describe-queue --instance-id <instance-id> --queue-id <queue-id>

# Routing profile
aws connect describe-routing-profile --instance-id <instance-id> --routing-profile-id <profile-id>

# Hours of operation
aws connect describe-hours-of-operation --instance-id <instance-id> --hours-of-operation-id <id>

# Phone number details
aws connect describe-phone-number --phone-number-id <phone-number-id>

# Lambda function associations
aws connect list-lambda-functions --instance-id <instance-id>

# Lex bot associations
aws connect list-lex-bots --instance-id <instance-id>
```

Read `references/connect-guardrails.md` before concluding on any Connect issue.

### Step 3 — Detailed path (low-confidence cases only)

```
# Contact trace records (via Kinesis or S3)
aws s3 ls s3://<ctr-bucket>/connect/<instance-name>/ContactTraceRecords/

# CloudTrail for API-level events
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=connect.amazonaws.com

# CloudWatch metrics for telephony
aws cloudwatch get-metric-statistics --namespace AWS/Connect --metric-name CallsPerInterval ...
aws cloudwatch get-metric-statistics --namespace AWS/Connect --metric-name MisconfiguredPhoneNumbers ...

# Real-time metrics for queue health
aws connect get-current-metric-data --instance-id <instance-id> \
  --filters '{"Queues":["<queue-arn>"]}' \
  --current-metrics '[{"Name":"OLDEST_CONTACT_AGE","Unit":"SECONDS"},{"Name":"CONTACTS_IN_QUEUE","Unit":"COUNT"}]'
```

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-instance` | Instance config, status, features |
| `list-instance-attributes` | Enabled features (contact flow logs, streaming, etc.) |
| `describe-contact` | Specific contact details, disconnect reason |
| `describe-contact-flow` | Contact flow definition and metadata |
| `describe-queue` | Queue config, outbound caller ID, hours |
| `describe-routing-profile` | Agent routing config, queue priorities |
| `describe-hours-of-operation` | Business hours configuration |
| `describe-phone-number` | Phone number status, target flow |
| `get-current-metric-data` | Real-time queue and agent metrics |
| `get-metric-data-v2` | Historical metrics and analytics |
| `list-lambda-functions` | Lambda integrations |
| `list-lex-bots` | Lex bot integrations |
| `search-contacts` | Search contacts by criteria |
| `describe-user` | Agent user configuration |
| `describe-user-hierarchy-group` | Agent hierarchy |

## Gotchas: Amazon Connect

These are the mistakes commonly made during Connect troubleshooting.

- Contact flow logs must be explicitly ENABLED on the instance AND on each contact flow. Enabling on the instance alone is not enough.
- Contact flows have a 32 KB size limit for the flow definition. Complex flows hit this limit and fail to publish.
- Lambda functions invoked from contact flows have an 8-second timeout. This is NOT configurable — it's a hard Connect limit regardless of the Lambda function's own timeout setting.
- DTMF input is only captured during "Get customer input" blocks. "Play prompt" blocks do NOT capture DTMF.
- Queue transfer vs direct transfer: queue transfer puts the contact in a queue for the next available agent. Direct transfer sends to a specific agent or external number.
- Agents must be in "Available" status to receive contacts. "Routable" custom statuses do NOT route contacts unless explicitly configured.
- CCP (Contact Control Panel) requires WebRTC. It needs specific ports and domains whitelisted. Corporate firewalls and VPNs commonly block CCP connectivity.
- Phone numbers are region-specific. A number claimed in us-east-1 cannot be used by an instance in eu-west-2.
- Outbound calls require a caller ID number that is claimed in the Connect instance. You cannot spoof arbitrary caller IDs.
- Contact trace records (CTRs) are delivered asynchronously. They may take up to 24 hours to appear in the default reporting. Use Kinesis streaming for near-real-time CTR delivery.
- Historical metrics have a 24-hour delay for some aggregations. Real-time metrics are available immediately but only show current state.
- Connect uses a soft phone (CCP) by default. Desk phones are supported but add latency and are subject to PSTN quality.
- Quick connects define transfer targets. Without a quick connect, agents cannot transfer calls to that destination.
- Contact attributes are key-value pairs passed through the contact flow. They have a 32 KB total size limit per contact.
- Lex bots in contact flows use streaming for voice. The Lex bot must be in the same region as the Connect instance.

### Contact flow block types

| Block | Purpose |
|-------|---------|
| Play prompt | Play audio or TTS (no input capture) |
| Get customer input | Capture DTMF or Lex intent |
| Set working queue | Set the queue for transfer |
| Transfer to queue | Route contact to a queue |
| Transfer to phone number | Transfer to external number |
| Transfer to flow | Transfer to another contact flow |
| Invoke AWS Lambda | Call Lambda (8s timeout) |
| Set contact attributes | Set key-value pairs on contact |
| Check contact attributes | Branch on attribute values |
| Check hours of operation | Branch on business hours |
| Check queue status | Branch on queue metrics |
| Loop | Repeat a section |
| Disconnect / hang up | End the contact |

### Agent states

| State | Receives contacts? |
|-------|--------------------|
| Available | Yes |
| On call | No (already on contact) |
| After call work (ACW) | No |
| Offline | No |
| Custom (routable) | Only if configured |
| Custom (non-routable) | No |
| Missed | No (auto-set after missed contact) |
| Error | No |

### Disconnect reasons

| Reason | Meaning |
|--------|---------|
| CUSTOMER_DISCONNECT | Customer hung up |
| AGENT_DISCONNECT | Agent hung up |
| THIRD_PARTY_DISCONNECT | Third party hung up |
| TELECOM_PROBLEM | Telephony issue |
| CONTACT_FLOW_DISCONNECT | Flow ended the contact |
| OTHER | Unclassified |

## Gotchas: CCP and Connectivity

- CCP requires WebRTC with TURN/STUN. Ports 3478 (UDP/TCP) must be open.
- CCP requires access to *.awsapps.com, *.amazonaws.com, and *.cloudfront.net domains.
- VPN split tunneling can cause CCP audio issues. Media traffic should not go through VPN.
- Browser requirements: Chrome or Firefox latest. Safari and Edge have limited support.
- CCP softphone uses opus codec. Network jitter > 30ms or packet loss > 0.5% causes audible quality issues.
- Multiple CCP tabs cause agent state conflicts. Only one CCP session per agent.

## Gotchas: Telephony

- Toll-free numbers have per-minute costs for inbound calls. DID numbers have monthly fees.
- Number porting takes 2-4 weeks and requires LOA (Letter of Authorization) from the current carrier.
- Emergency calling (E911) requires specific configuration and is not automatic.
- International calling must be explicitly enabled per country and has separate rate tables.
- Outbound calling campaigns require specific compliance configurations (TCPA, etc.).

## Anti-hallucination rules

1. Always cite specific contact flow logs, CTR data, CloudWatch metrics, or API responses as evidence.
2. Lambda timeout in Connect is 8 seconds. Never claim it can be increased beyond 8 seconds.
3. Contact flow logs require enablement at BOTH instance and flow level. Never claim instance-level enablement is sufficient.
4. DTMF is only captured in "Get customer input" blocks. Never claim other blocks capture DTMF.
5. CTRs can take up to 24 hours. Never claim CTRs are available immediately without Kinesis streaming.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 36 runbooks

Runbooks are organized by failure domain. Use the appropriate runbook based on the symptom category.

| Category | IDs | Covers |
|----------|-----|--------|
| A — Contact Flow | A1-A4 | Flow errors, publish failures, loop/timeout, attribute issues |
| B — Call Routing | B1-B4 | Queue routing failures, hours of operation, priority/overflow, skills-based routing |
| C — Agent Issues | C1-C4 | Login failures, state problems, missed contacts, CCP connectivity |
| D — Telephony | D1-D4 | Inbound call failures, outbound call failures, call quality, DTMF issues |
| E — Integrations | E1-E4 | Lambda timeout/errors, Lex bot failures, Kinesis streaming, S3 recording |
| F — Channels | F1-F3 | Chat issues, task channel issues, multi-channel routing |
| G — Metrics & Reporting | G1-G3 | Missing CTRs, real-time metrics gaps, historical reporting issues |
| H — Security & Access | H1-H3 | SAML/SSO failures, IAM permissions, security profile issues |
| I — Phone Numbers | I1-I3 | Number claiming failures, porting issues, caller ID problems |
| Z — Catch-All | Z1 | General troubleshooting |
