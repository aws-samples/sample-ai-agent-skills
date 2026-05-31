# Health Dashboard Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Health Dashboard problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Health Dashboard when the console alone isn't enough — event interpretation, affected resource identification, scheduled change management, EventBridge integration, notification delivery, organizational health view, delegated administration, Health API access, event filtering, incident response, and maintenance planning.

### Activate When

- Confusion about health event meaning or impact
- Need to identify affected resources for an event
- Scheduled maintenance planning and coordination
- EventBridge not receiving health events
- Health notifications not being delivered
- Organizational health view not working
- Delegated admin setup for health
- Health API access errors
- Event filtering not returning expected results
- Incident response coordination needed
- Maintenance window planning required

---

## Skill Structure

```
healthdashboard-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-event-interpretation.md
    ├── A2-affected-resources.md
    ├── A3-scheduled-changes.md
    ├── B1-eventbridge-integration.md
    ├── B2-notification-setup.md
    ├── C1-organizational-health.md
    ├── C2-delegated-admin.md
    ├── D1-health-api.md
    ├── D2-event-filtering.md
    ├── E1-incident-response.md
    ├── E2-maintenance-planning.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (16 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Events** | A1–A3 | Event interpretation, affected resources, scheduled changes |
| **B — Notifications** | B1–B2 | EventBridge integration, notification setup |
| **C — Organization** | C1–C2 | Organizational health, delegated admin |
| **D — API** | D1–D2 | Health API, event filtering |
| **E — Response** | E1–E2 | Incident response, maintenance planning |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
