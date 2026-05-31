# Amazon Connect Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Connect problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Connect when the console alone isn't enough — contact flow errors, call routing failures, agent state issues, telephony quality problems, CCP connectivity, Lambda/Lex integration failures, CTR data gaps, chat/task channel issues, and phone number management.

### Activate When

- Calls are not routing correctly or going to wrong queues
- Agents cannot log in or receive contacts
- Contact flows have errors or fail to publish
- DTMF or IVR is not working
- Call quality is poor (latency, jitter, packet loss)
- Outbound calls fail
- Queues are overflowing or contacts are stuck
- CTR data is missing or delayed
- Lex bots are not responding in contact flows
- Lambda integrations fail or timeout within contact flows
- Real-time or historical metrics are incorrect
- Chat or task channels have issues
- SAML/SSO login fails for agents or admins
- CCP (Contact Control Panel) connectivity problems
- Phone number porting or claiming issues
- The user says something is wrong with Connect

---

## Skill Structure

```
connect-troubleshooting/
├── SKILL.md                          # Main skill definition and investigation workflow
├── README.md                         # This file
└── references/
    ├── A1-contact-flow-errors.md
    ├── A2-flow-publish-failures.md
    ├── A3-flow-loop-timeout.md
    ├── A4-contact-attribute-issues.md
    ├── B1-queue-routing-failures.md
    ├── B2-hours-of-operation.md
    ├── B3-priority-overflow.md
    ├── B4-skills-based-routing.md
    ├── C1-agent-login-failures.md
    ├── C2-agent-state-problems.md
    ├── C3-missed-contacts.md
    ├── C4-ccp-connectivity.md
    ├── D1-inbound-call-failures.md
    ├── D2-outbound-call-failures.md
    ├── D3-call-quality.md
    ├── D4-dtmf-issues.md
    ├── E1-lambda-integration.md
    ├── E2-lex-bot-failures.md
    ├── E3-kinesis-streaming.md
    ├── E4-s3-recording.md
    ├── F1-chat-issues.md
    ├── F2-task-channel-issues.md
    ├── F3-multi-channel-routing.md
    ├── G1-missing-ctrs.md
    ├── G2-real-time-metrics.md
    ├── G3-historical-reporting.md
    ├── H1-saml-sso-failures.md
    ├── H2-iam-permissions.md
    ├── H3-security-profile-issues.md
    ├── I1-number-claiming-failures.md
    ├── I2-number-porting.md
    ├── I3-caller-id-problems.md
    ├── Z1-general-troubleshooting.md
    ├── connect-guardrails.md
    └── connect-hallucination-patterns.yaml
```

---

## Runbook Library (36 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Contact Flow** | A1–A4 | Flow errors, publish failures, loop/timeout, attribute issues |
| **B — Call Routing** | B1–B4 | Queue routing failures, hours of operation, priority/overflow, skills-based routing |
| **C — Agent Issues** | C1–C4 | Login failures, state problems, missed contacts, CCP connectivity |
| **D — Telephony** | D1–D4 | Inbound call failures, outbound call failures, call quality, DTMF issues |
| **E — Integrations** | E1–E4 | Lambda timeout/errors, Lex bot failures, Kinesis streaming, S3 recording |
| **F — Channels** | F1–F3 | Chat issues, task channel issues, multi-channel routing |
| **G — Metrics & Reporting** | G1–G3 | Missing CTRs, real-time metrics gaps, historical reporting issues |
| **H — Security & Access** | H1–H3 | SAML/SSO failures, IAM permissions, security profile issues |
| **I — Phone Numbers** | I1–I3 | Number claiming failures, porting issues, caller ID problems |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails & Anti-Hallucination

### Connect Guardrails (`connect-guardrails.md`)
12 rules covering: Lambda 8-second timeout, contact flow log enablement, DTMF capture blocks, CTR delivery latency, CCP WebRTC requirements, contact attribute size limits, agent state routing behavior, phone number region binding, Lex bot region matching, quick connect requirements, contact flow size limits, and outbound caller ID requirements.

### Hallucination Patterns (`connect-hallucination-patterns.yaml`)
10 patterns that LLMs commonly get wrong about Amazon Connect, including:
- Claiming Lambda timeout in contact flows is configurable (it's fixed at 8 seconds)
- Claiming contact flow logs are enabled by default (they require explicit enablement)
- Confusing "Play prompt" with "Get customer input" for DTMF capture
- Claiming CTRs are available immediately (up to 24 hours without Kinesis)
- Assuming agents in any custom status receive contacts

---

## Investigation Workflow

Each runbook follows a consistent phased structure:

### Phase 1 — Triage
Collect initial evidence using Connect APIs, contact flow logs, CloudWatch metrics, and CTR data. Classify the failure domain.

### Phase 2 — Enrich / Remediate
Deep dive into the specific domain using contact flow analysis, queue configuration review, agent state inspection, telephony diagnostics, or integration testing.

### Phase 3 — Report
State root cause with evidence, blast radius, severity, and recommended mitigations (immediate and long-term).

### Output Format
Every runbook produces structured YAML output:
```yaml
root_cause: "<category> — <detail>"
evidence:
  - type: <source>
    content: "<specific finding>"
severity: CRITICAL | HIGH | MEDIUM
mitigation:
  immediate: "<action>"
  long_term: "<prevention>"
```

---

## Prerequisites

- AWS CLI or SDK access with Amazon Connect, CloudWatch, CloudWatch Logs, Lambda, Lex, Kinesis, S3, and IAM permissions
- For contact flow debugging: contact flow logs enabled on the instance and flows
- For telephony diagnostics: CloudWatch metrics and CTR access
- For CCP debugging: browser developer tools and network diagnostics
- For integration debugging: Lambda, Lex, Kinesis, and S3 permissions

---

## Usage Examples

### Calls Not Routing
```
Calls to our main number are going to voicemail instead of agents.
Check the contact flow, hours of operation, queue configuration,
and agent availability. Agents say they are available.
```

### CCP Audio Issues
```
Agents report one-way audio or choppy calls on the CCP. They are
on VPN. Check CCP connectivity requirements, WebRTC ports, and
network quality metrics.
```

### Lambda Integration Failing
```
Our contact flow calls a Lambda function to look up customer data
but it keeps timing out. The Lambda function works fine when tested
directly. Check the Connect Lambda timeout limit and function
execution time.
```

### Missing Contact Records
```
We can't find CTRs for calls from yesterday afternoon. Check if
CTR streaming is configured, if the Kinesis stream or S3 delivery
is working, and if there's a delay.
```

### General Triage
```
Something is wrong with our Connect instance but I'm not sure what.
Agents are complaining about various issues. Run a general
investigation — check instance status, contact flows, queues,
agent states, and follow whichever runbook matches the symptoms.
```

---

## License

MIT-0
