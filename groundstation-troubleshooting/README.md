# AWS Ground Station Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Ground Station problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Ground Station — satellite contacts, antenna configuration, data delivery, mission profiles, and ground station locations.

### Activate When

- Contact scheduling or reservation failures
- Antenna configuration or tracking errors
- Data delivery failures to EC2 or S3
- Mission profile configuration problems
- Ground station location or availability issues
- Satellite onboarding problems
- General Ground Station errors

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Contacts** | A1–A2 | Scheduling failures, contact execution |
| **B — Antenna** | B1–B2 | Configuration errors, tracking issues |
| **C — Data Delivery** | C1–C2 | Dataflow endpoints, S3 delivery |
| **D — Mission Profiles** | D1–D2 | Profile configuration, validation |
| **E — Ground Stations** | E1–E2 | Location selection, availability |
| **Z — Catch-All** | Z1–Z2 | General troubleshooting, satellite config |

---

## Prerequisites

- AWS CLI v2 with GroundStation permissions
- Permissions: `groundstation:Get*`, `groundstation:List*`, `groundstation:Describe*`

---

## License

MIT-0
