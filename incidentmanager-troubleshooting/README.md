# Incident Manager Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Systems Manager Incident Manager problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Incident Manager when the console alone isn't enough — response plan configuration, incident creation and lifecycle, escalation plans, contact channels, automation runbooks, timeline events, metrics integration, CloudWatch alarm triggers, EventBridge integration, post-incident analysis, and action item tracking.

### Activate When

- Response plan creation or configuration fails
- Incidents not being created from alarms
- Escalation plan not engaging correct contacts
- Contact channels not delivering notifications
- Engagement plans not activating
- Automation runbooks failing during incidents
- Manual steps not appearing or completing
- Timeline events missing or incorrect
- Metrics not showing in incident dashboard
- CloudWatch alarm not triggering response plan
- EventBridge rules not creating incidents
- Post-incident analysis not available
- Action items not tracking properly

---

## Skill Structure

```
incidentmanager-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-response-plan.md
    ├── A2-incident-creation.md
    ├── A3-escalation.md
    ├── B1-contact-channels.md
    ├── B2-engagement-plans.md
    ├── C1-automation-runbooks.md
    ├── C2-manual-steps.md
    ├── D1-timeline-events.md
    ├── D2-metrics.md
    ├── E1-cloudwatch-alarms.md
    ├── E2-eventbridge.md
    ├── F1-analysis.md
    ├── F2-action-items.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (20 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Response** | A1–A3 | Response plan, incident creation, escalation |
| **B — Contacts** | B1–B2 | Contact channels, engagement plans |
| **C — Runbooks** | C1–C2 | Automation runbooks, manual steps |
| **D — Timeline** | D1–D2 | Timeline events, metrics |
| **E — Integration** | E1–E2 | CloudWatch alarms, EventBridge |
| **F — Post-Incident** | F1–F2 | Analysis, action items |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
